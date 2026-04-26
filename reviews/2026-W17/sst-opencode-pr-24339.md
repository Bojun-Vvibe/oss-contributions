---
pr: 24339
repo: sst/opencode
sha: dfca395dd40fb26043555e090b754b3acd196743
verdict: merge-after-nits
date: 2026-04-26
---

# sst/opencode #24339 — fix(mcp): stabilize tool ordering

- **Author**: pascalandr
- **Head SHA**: dfca395dd40fb26043555e090b754b3acd196743
- **Link**: https://github.com/sst/opencode/pull/24339
- **Size**: ~80 diff lines across `packages/opencode/src/mcp/index.ts` and `packages/opencode/test/mcp/lifecycle.test.ts`.

## Scope

Makes the order of MCP tools exposed to the model deterministic. Previously the order depended on (a) hash-map iteration order over `s.clients`, and (b) `Effect.forEach(..., { concurrency: "unbounded" })` interleaving — so prompt-cache invalidation, model behavior, and replay diffs would all fluctuate run-to-run with no real change. Fix sorts both the per-client tool list and the inter-client iteration alphabetically.

## Specific findings

- `packages/opencode/src/mcp/index.ts:117-119` — new `byName` comparator using `localeCompare`. Reasonable default; `localeCompare` is locale-sensitive in theory but for ASCII tool names this is a non-issue. If two MCP servers ever ship tools that differ only by Unicode normalisation, the result could vary between machines, but that's a contrived case.
- `packages/opencode/src/mcp/index.ts:157` — `result.tools.toSorted(byName)` applied at the cache-population layer in `defs()`. Good — sorts at the source so the cache itself is canonical, not just the read path.
- `packages/opencode/src/mcp/index.ts:629-651` — the bigger behavioral change: `connectedClients` is now `.toSorted(([a], [b]) => a.localeCompare(b))`, **and** the previous `Effect.forEach(..., { concurrency: "unbounded" })` was replaced with a plain `for...of`. The PR title says "stabilize tool ordering" but the concurrency drop is a separate observable change — tool conversion is now sequential across servers. For typical MCP setups (≤ 10 servers, ≤ 100 tools each, all in-memory `convertMcpTool` calls with no I/O) this is fine and arguably preferable for log readability, but it deserves a one-line note in the PR body. If any future MCP variant does I/O inside `convertMcpTool` this becomes a perf cliff.
- `packages/opencode/src/mcp/index.ts:649` — the per-server inner loop also re-sorts via `listed.toSorted(byName)`. This is redundant with the sort at `:157` (the cache should already be sorted) but it's defense-in-depth and cheap. Fine to keep.
- `packages/opencode/test/mcp/lifecycle.test.ts:236-265` — single deterministic test asserting `["a-server_alpha", "a-server_beta", "z-server_zeta"]`. Pins both axes (inter-server + intra-server) in one shot. Good test.

## Risk

Low. The only behavioral change beyond ordering is the loss of unbounded concurrency in tool registration; for a synchronous body this is invisible. The deterministic ordering will improve prompt-cache hit rates noticeably for users with MCP setups.

## Verdict

**merge-after-nits** — call out the concurrency change in the PR body (or split it into a follow-up if the maintainer prefers strict-scope PRs). The functional change itself is well-tested and correct.
