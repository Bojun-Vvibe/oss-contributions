# charmbracelet/crush #2750 — fix(lsp): score-based client selection so catch-all LSPs don't steal files

- **PR:** charmbracelet/crush#2750
- **Head SHA:** `92b90311ecd36c82c0967e09c9777f66741145a6`
- **Files:** 4 changed (+180 / -11)

## What changed

- `internal/lsp/client.go:347-376` introduces `(*Client).MatchScore(path string) int` with a documented three-tier scale: `0` (outside workspace, or `FileTypes` set but no match), `1` (inside workspace, no `FileTypes` configured = catch-all), `2` (inside workspace and `FileTypes` explicitly names the extension/detected language). The old `HandlesFile` becomes a thin `MatchScore(path) > 0` wrapper, preserving every existing call site (this is the load-bearing compatibility detail — `lsp_diagnostics`-style broadcast tools that want "anyone willing" still get the old semantics for free).
- `internal/lsp/manager.go:78-110` adds `(*Manager).BestClientFor(path string) *Client`, the new pick-one API: iterates `m.clients`, takes the highest non-zero `MatchScore`, and on tie picks the alphabetically smaller `name` (deterministic). This is the API surface for tools like `lsp_references` that have to commit to a single client.
- `internal/agent/tools/references.go:103` is the consumer migration. The previous 7-line loop that took the first client in `Clients().Seq()` whose `HandlesFile(absPath)` returned true is replaced by a one-liner `client := lspManager.BestClientFor(absPath)`. The previous behaviour was non-deterministic because `csync.Map` iteration order is randomised — when a user had both a Python LSP and any catch-all client (e.g. a misconfigured `extensions:`-vs-`filetypes:` typo silently zeroing FileTypes), `references` would land on either depending on map iteration luck.
- New `internal/lsp/best_client_test.go` (+122) is unusually thorough for this kind of fix: `TestMatchScore` covers all five cells of the score matrix; `TestBestClientForPrefersSpecificMatchOverCatchAll` runs the assertion in a `for range 100` loop specifically to catch the original race-condition bug class; `TestBestClientForFallsBackToCatchAll` confirms catch-all still wins when nothing is more specific; `TestBestClientForDeterministicTiebreak` validates the alphabetical-name tiebreak across 100 iterations (so flake risk is essentially zero).

## Risks / notes

- The 100-iteration loops in the tests are a deliberate counter to map-iteration-order flakiness — a reasonable choice, but they do quietly expand the test runtime. If the manager grows expensive setup later, this pattern may need to migrate to a deterministic ordering in `Clients().Seq()` itself rather than 100x iteration in tests. Not actionable today.
- `BestClientFor` doesn't surface *which* client lost; if a user's catch-all is repeatedly losing to a more specific client they'd want, there's no diagnostic. A `slog.Debug` line listing scores would help debugging without affecting hot paths.
- The migration only updates `references.go`. `grep`ping for other consumers of the old "iterate Clients() and break on first HandlesFile" pattern would be a 5-minute follow-up to make sure no other tool is silently still racing — `definition`, `hover`, `signature_help`, `implementations`, `type_definition`, `prepare_rename`, etc. all share the same shape.
- The doc-comment on `MatchScore` correctly distinguishes "broadcast" (use `HandlesFile`) from "pick-one" (use `BestClientFor`) — that's exactly the kind of guidance future contributors need, very nicely done.

## Verdict

**merge-as-is** — the bug class (race-condition routing for pick-one tools) is real and this is the cleanest possible fix: additive API, full back-compat for broadcast callers, deterministic tiebreak, and the test suite specifically exercises the original failure mode. A follow-up to migrate other pick-one LSP consumers (definition, hover, etc.) would be valuable but not required for this PR.
