# sst/opencode #25077 — fix: invert *_ready getters to fix server status indicator

- **Author:** Brendonovich (Brendan Allan)
- **SHA:** `b72e423`
- **State:** MERGED
- **Size:** +3 / -3 across 1 file (`packages/app/src/context/global-sync/child-store.ts`)
- **Verdict:** `merge-as-is`

## Summary

Three-line polarity fix on `provider_ready` / `mcp_ready` / `lsp_ready` getters in
`packages/app/src/context/global-sync/child-store.ts:189,210,216`. The getters previously
returned `providerQuery.isLoading` / `mcpQuery.isLoading` / `lspQuery.isLoading` as the
"ready" flag, which is the literal inverse of the intended semantics — the status
indicator showed "ready" while the query was still loading and "not ready" once it had
resolved. The fix prefixes each return with `!`, making "ready ↔ not loading" hold.

## Reasoning

The bug is a classic boolean-polarity inversion that escaped review because the names
look adjacent in editors (`isLoading` / `_ready`). The fix is the minimal correct change
— no rename, no helper introduced, no consumer surface change. Because the `*_ready`
getters are evaluated lazily, the consumers reading them (the server status indicator
UI mentioned in the PR title) immediately see the corrected polarity once the store
remounts; no migration needed. Worth confirming there's no downstream code that had
silently adapted to the inverted reading (e.g. a UI that displays "Ready" while
loading by reading `*_ready` and showing it inverted in JSX), but a quick `rg
'_ready'` over the consuming packages would surface that and the PR is small enough
that the maintainer can verify in one pass. The companion `mcp` / `lsp` getters at
`:213,219` correctly already use `mcpQuery.isLoading ? {} : ...` ternary — those
weren't broken, only the boolean flags were.
