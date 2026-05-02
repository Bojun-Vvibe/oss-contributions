# charmbracelet/crush #2750 — fix(lsp): score-based client selection so catch-all LSPs don't steal files

- **Repo:** charmbracelet/crush
- **PR:** #2750
- **Head SHA:** `92b90311ecd36c82c0967e09c9777f66741145a6`
- **Verdict:** merge-as-is

## Summary

Likely fixes #1751. Diagnosis is the strongest part: empty `FileTypes`
silently becomes a catch-all (`handlesFiletype` returns true on
`len(fileTypes) == 0`), and Go map iteration in `lsp_references` then
non-deterministically picks one of the catch-all clients to handle
every file. On the reporter's machine the CSS LSP consistently won.

Fix replaces first-match-wins with a scoring API:

- `Client.MatchScore(path) int` in `internal/lsp/client.go:347-377`:
  `0` outside workspace or explicit-mismatch, `1` for catch-all
  inside workspace, `2` for explicit FileTypes match.
  `HandlesFile` is rewritten as `MatchScore > 0` — broadcast tools
  (`lsp_diagnostics`, `notifyLSPs`) keep current behavior.
- `Manager.BestClientFor(path) *Client` in
  `internal/lsp/manager.go:91-107`: highest score wins, ties broken
  by name for determinism.
- `internal/agent/tools/references.go:103` swaps the racey
  iterate-and-break for `lspManager.BestClientFor(absPath)`.

## Specific notes

- **`client.go:374-377`:** `handlesFiletype(c.name, c.fileTypes, path)`
  returns boolean → score 2. Clean two-tier design and easy to extend
  to e.g. score 3 for content-type detection later.
- **`manager.go:101-105`:** `score > bestScore || (score == bestScore && (best == nil || name < bestName))` — `best == nil` defensive check is redundant once `bestScore > 0` (we'd have set `best` then), but harmless. Reads fine.
- **`best_client_test.go:65-78`:** the 100-iteration loop in
  `TestBestClientForPrefersSpecificMatchOverCatchAll` is exactly the
  right shape for catching map-order leaks. Deterministic-tiebreak
  test (lines 117-122) does the same for ties.
- **Scope discipline:** PR explicitly leaves the `extensions` vs
  `filetypes` config schema mismatch and the silent-empty-FileTypes
  warning to follow-ups. Right call — fixing both in one PR would
  bundle unrelated risk.

## Rationale

Excellent root-cause analysis, minimal-surface-area fix, deterministic
behavior under repeat runs. No nits worth blocking on.
