# charmbracelet/crush #2750 — fix(lsp): score-based client selection so catch-all LSPs don't steal files

- **Repo:** charmbracelet/crush
- **PR:** https://github.com/charmbracelet/crush/pull/2750
- **HEAD SHA:** `92b90311ecd36c82c0967e09c9777f66741145a6`
- **Author:** georgeglarson
- **Verdict:** `merge-after-nits`

## What the diff does

Closes #1751-style symptom where a catch-all LSP client (one with
empty `FileTypes`, often the result of a config typo like
`extensions:` instead of `filetypes:` causing the field to be
silently dropped) would intercept files that an explicitly-claiming
client should have served. The bug surfaced as flaky LSP behavior
because Go's `map` iteration order is randomized — same workspace,
different choice each invocation.

Three-file change:

1. `internal/lsp/client.go:345-376` — refactors `HandlesFile` to
   `HandlesFile() bool { return c.MatchScore(path) > 0 }` and
   introduces new `MatchScore(path) int` returning a tri-state:
   - `0` — outside workspace OR `FileTypes` set but no match
   - `1` — inside workspace, no `FileTypes` (catch-all)
   - `2` — inside workspace, `FileTypes` explicitly names the
     extension or detected language

2. `internal/lsp/manager.go:78-108` — new
   `BestClientFor(path) *Client` iterates `s.clients.Seq2()` and
   returns the highest-scoring client, breaking ties alphabetically
   by name (`name < bestName`) so the choice is deterministic
   across runs. Returns nil when no client has score > 0.
   Docstring at `:78-90` explicitly distinguishes "tools that pick
   a single client (lsp_references) use this" vs. "tools that
   broadcast (lsp_diagnostics, notifyLSPs) keep using HandlesFile".

3. `internal/agent/tools/references.go:103` — replaces the old
   first-match-wins `for c := range lspManager.Clients().Seq()`
   loop with `client := lspManager.BestClientFor(absPath)`.

Tests at `internal/lsp/best_client_test.go` (+122 lines) cover:

- `TestMatchScore` parametrized table: explicit ext match (score 2),
  empty filetypes catch-all (score 1), explicit but mismatched
  (score 0), outside workspace (score 0), nil client (score 0).
- `TestBestClientForPrefersSpecificMatchOverCatchAll` — the
  load-bearing regression test, runs the assertion 100 times in a
  loop to catch the map-iteration-order race.
- `TestBestClientForFallsBackToCatchAll` — locks the property
  that catch-all is still used when no explicit client claims.
- `TestBestClientForReturnsNilWhenNothingHandlesPath` — locks the
  nil return.
- `TestBestClientForDeterministicTiebreak` — two equally-specific
  clients, asserts alphabetical tiebreak picks the smaller name
  100 times.

## Why it's right

The diagnosis is exact. Go map iteration order has been intentionally
randomized since 1.0 specifically to discourage code from relying on
it; the prior `for c := range Clients().Seq()` with first-match-wins
was that exact anti-pattern. The map randomization is a feature, not
a bug — but it surfaced this latent ambiguity in the LSP routing
table.

The tri-state scoring is the right shape for the problem. The bug
was that "willing to handle" was binary — both the catch-all and the
explicit-match returned `true`, and the iteration order picked the
loser at random. By splitting "willing" (any score > 0) from
"strongly claims" (score 2 > score 1), the primitive that "must
pick one" tools need (`BestClientFor`) has unambiguous semantics
without breaking the primitive that "broadcast to all" tools need
(`HandlesFile` is unchanged because it still returns true on any
non-zero score).

The deterministic tiebreak via alphabetical name comparison
(`name < bestName` at `manager.go:99`) is the small but important
detail — without it, two clients with `FileTypes: [py]` would still
race on map iteration order. Alphabetical is a stable, explainable,
config-controllable order; users who want a specific tiebreak can
rename their LSP entry.

The 100-iteration assertion loop in
`TestBestClientForPrefersSpecificMatchOverCatchAll` (`:80`) is
exactly the right test shape for a map-iteration-order race —
asserting once would pass on the first try if the iteration happened
to put the right client first; asserting 100 times effectively
guarantees the bad branch would be exercised at least once if it
existed.

The Go docstring at `manager.go:78-90` explicitly names the
broadcast-vs-single-pick distinction (`lsp_diagnostics`,
`notifyLSPs` keep using `HandlesFile`; only `lsp_references` uses
`BestClientFor`), which is the load-bearing comment because a
future cleanup might naively migrate all callers to `BestClientFor`
and silently break diagnostic broadcast.

## Nits

1. **`HandlesFile` is now a thin wrapper around `MatchScore`** but
   external callers (other tools) might find it confusing that
   the same workspace+ext path can return `true` from
   `HandlesFile()` but be passed over by `BestClientFor()` when a
   higher-scoring client exists. A docstring on `HandlesFile`
   noting "use BestClientFor when picking a single client" would
   prevent the next refactor from deleting `HandlesFile` and
   forcing all callers into `BestClientFor`.

2. **Tiebreak by name is config-controllable but undocumented for
   users.** The user-facing config docs should mention that
   "if two LSPs claim the same file type, the one with the
   alphabetically earlier name wins" so users hit by the
   tiebreak know how to influence it (rename one LSP).

3. **`MatchScore` returns `int` rather than a typed enum** —
   `Score(0|1|2)` is small enough that a typed `MatchLevel` enum
   with named constants (`MatchNone`, `MatchCatchAll`,
   `MatchExplicit`) would self-document without runtime cost.
   Especially valuable if a fourth level is ever added (e.g.
   "explicit *and* highest-priority alias match" → score 3).

4. **No test for the `lsp_references` integration** — the file
   `internal/agent/tools/references.go` is the only caller in this
   diff, but there's no test that asserts `find()` actually picks
   the right client when both candidates exist in the manager.
   A table-test in `references_test.go` constructing a manager
   with one explicit + one catch-all client and asserting `find()`
   uses the explicit one would lock the end-to-end property the
   PR exists to deliver.

5. **`BestClientFor` walks `Clients().Seq2()` linearly per call.**
   For a small N (most users have ≤5 LSP clients) this is fine,
   but `lsp_references` may be invoked in a hot loop during agent
   reasoning. If the result is stable across the workspace
   lifetime (it should be, since `FileTypes` and `cwd` are
   config-time), the manager could memoize per-extension. Not
   a blocker — premature optimization for a non-hot path.

6. **`MatchScore` `nil` check at `:359-361`** returns 0 silently.
   A debug log would help diagnose "why is this client being
   skipped?" cases — the existing
   `slog.Debug("File outside workspace", ...)` at `:362` is the
   right shape, but the nil case has no breadcrumb at all.

## Verdict rationale

Right diagnosis (Go map iteration order surfaced a latent ambiguity
in LSP routing), right primitive (tri-state scoring with explicit
"single pick" vs. "broadcast" callers), right test surface
(100-iteration loop on the canonical race plus deterministic
tiebreak verification). Documentation nits on `HandlesFile`,
user-facing tiebreak rule, and the `MatchScore` typed-enum
opportunity; no integration test on the `lsp_references` end-to-end
path that's the PR's whole point.

`merge-after-nits`
