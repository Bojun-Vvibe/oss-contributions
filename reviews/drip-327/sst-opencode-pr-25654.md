# sst/opencode #25654 — fix(mcp): ensure Accept header includes both required values for Streamable HTTP

- SHA: `bdb7d1cd852838057d58690b9c8c947c1bbfeb9e`
- State: OPEN, +75/-2 across 2 files
- Files: `packages/opencode/src/mcp/index.ts`, `packages/opencode/test/mcp/headers.test.ts`

## Summary

Adds a custom `mcpFetch` wrapper passed into `StreamableHTTPClientTransport` that guarantees the `Accept` header carries both `application/json` and `text/event-stream`, per the MCP Streamable HTTP spec. Some servers (Zhipu/BigModel called out) reject otherwise. Test file is extended to match the new `fetch` option in transport calls.

## Notes

- `packages/opencode/src/mcp/index.ts:303-313`: implementation is minimal — clones headers, checks for both substrings, only overwrites when one is missing. Good behavior; preserves user-supplied `accept` if already compliant.
- The substring check `accept.includes("application/json")` will also match `application/json+something` which is fine in practice but slightly loose; acceptable for an interop fix.
- Test diff at `packages/opencode/test/mcp/headers.test.ts:6-...` only widens the captured `options` shape to include `fetch?: unknown`. Would be stronger if a test asserted that the wrapped fetch actually injects the header when called with no `accept`. Nit, not a blocker.
- Only applied to `StreamableHTTPClientTransport`, not the SSE fallback — correct, since SSE transport sets its own `Accept`.

## Verdict

`merge-after-nits` — add a unit test that exercises `mcpFetch` directly (header missing → set, header partial → overwrite, header complete → untouched).