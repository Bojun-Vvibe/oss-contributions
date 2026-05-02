# charmbracelet/crush PR #2750 — fix(lsp): score-based client selection so catch-all LSPs don't steal files

- **PR**: https://github.com/charmbracelet/crush/pull/2750
- **Head SHA**: `92b90311ecd36c82c0967e09c9777f66741145a6`
- **Size**: +180 / −11 across 4 files (`internal/agent/tools/references.go`, `internal/lsp/client.go`, `internal/lsp/manager.go`, new `internal/lsp/best_client_test.go`)
- **Verdict**: **merge-as-is**

## What changed

Three coordinated edits and one new test file:
1. `internal/lsp/client.go:347` introduces `MatchScore(path string) int` returning `0` (outside workspace OR explicit FileTypes mismatch), `1` (inside workspace, FileTypes empty → catch-all), or `2` (inside workspace, FileTypes explicitly matches). `HandlesFile` is reduced to `MatchScore(path) > 0` so existing broadcast callers (e.g. `lsp_diagnostics`) keep their willing-set semantics.
2. `internal/lsp/manager.go:78` adds `(*Manager).BestClientFor(path string) *Client`. It iterates `s.clients.Seq2()`, picks the highest-scoring willing client, and **breaks ties alphabetically by client name** — explicitly to defeat Go's randomized map iteration so the same workspace routes to the same client across runs.
3. `internal/agent/tools/references.go:103` replaces the prior `for c := range lspManager.Clients().Seq() { if c.HandlesFile(absPath) { client = c; break } }` first-match scan with a single `client := lspManager.BestClientFor(absPath)` call.
4. `internal/lsp/best_client_test.go` (new, +122) adds: `TestMatchScore` for the four score buckets (including the nil-receiver case → 0); `TestBestClientForPrefersSpecificMatchOverCatchAll` which loops 100 iterations to defeat map-order flakes; `TestBestClientForFallsBackToCatchAll`; `TestBestClientForReturnsNilWhenNothingHandlesPath`; and `TestBestClientForDeterministicTiebreak` (also 100 iterations) verifying alphabetical tiebreak.

## Why this matters

The bug is a real and subtle one: when a user mistypes `extensions:` instead of `filetypes:` in their LSP config, the field is silently dropped, leaving an LSP with `FileTypes=nil` (empty). The previous `HandlesFile` returned true for any file inside that LSP's cwd, so a misconfigured CSS LSP would happily claim a `.py` file. The first-match loop in `references.go` then returned whichever LSP `Clients().Seq()` happened to iterate first — Go randomizes that, so the same `.py` lookup would route to the Python LSP on one invocation and the CSS LSP on the next. The user-visible symptom is intermittent "No references found" or "wrong references found" with no error logged. Score-based selection (`2 > 1`) ensures the explicit Python LSP always wins over a catch-all; alphabetical tiebreak ensures determinism when two equally-specific clients claim the same path.

The split between `BestClientFor` (single-pick) and `HandlesFile` (broadcast) is the right API: tools like `lsp_diagnostics` legitimately want to ask every willing LSP, while `references` and similar single-answer tools need a winner.

## Specific call-outs

- The 100-iteration assertion loop in `TestBestClientForPrefersSpecificMatchOverCatchAll` is the right way to test against map-iteration randomization — a single assertion would have ~50% chance of accidentally passing the broken pre-fix code. Same for `TestBestClientForDeterministicTiebreak`. This is exactly the "stress the random surface" pattern these tests need.
- `MatchScore` returns `0` for nil receiver (line 351), correctly preventing nil-deref panics in callers that might iterate a stale snapshot. The test covers this.
- The doc comment on `BestClientFor` explicitly documents the contract for diagnostics-style callers ("Tools that broadcast … should iterate Clients() directly with HandlesFile") — this is exactly the right level of API guidance to leave for future maintainers.
- Doc comment on `MatchScore` enumerates the three score buckets clearly. Future scoring tweaks (e.g. adding language-detection match at score 3) will slot in naturally.
- `references.go` deletion of 8 lines + addition of 1 is the kind of cleanup the new abstraction earns. No behavior loss because the new helper subsumes both the loop and the willingness check.
- The tiebreak rationale ("Go's map iteration order is randomized, and without a tiebreak the same workspace would route to different clients on different invocations") is captured in both the doc comment and the test name. Good.
- No backward-compat concerns: `HandlesFile` keeps its previous external contract (`bool`, returns true iff willing).

## Verdict rationale

Textbook bug fix: clear root cause, minimal API addition, broadcast vs. single-pick separation done correctly, deterministic tiebreak, and tests that actually stress the failure mode. Nothing to push back on. Ship.
