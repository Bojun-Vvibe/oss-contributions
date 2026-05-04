# QwenLM/qwen-code PR #3818 — fix(core): coalesce MCP server rediscovery

- **Link:** https://github.com/QwenLM/qwen-code/pull/3818
- **Head SHA:** `f2e19a373c99b98aa4b3c64d7c9a75849d4565e0`
- **Author:** cyphercodes (Rayan Salhab)
- **Created:** 2026-05-04
- **Files:** `packages/core/src/tools/mcp-client-manager.test.ts` (+86/−0), `packages/core/src/tools/mcp-client-manager.ts` (+27/−0)
- **Diff size:** +113 / 0

## Summary
Prevents duplicate concurrent MCP-server-discovery work. Adds a `serverDiscoveryPromises: Map<string, Promise<void>>` field to `McpClientManager` and a public `discoverMcpToolsForServer` wrapper that:
1. If a discovery for the same `serverName` is already in flight, awaits it and returns.
2. Otherwise records the new promise, runs the existing logic (renamed to `discoverMcpToolsForServerInternal`), and removes itself from the map on completion via a `finally` block.
A targeted unit test verifies coalescing of two rediscovery calls into a single internal run, and that the map is cleaned up so a subsequent call performs real work.

## Specific citations
- `packages/core/src/tools/mcp-client-manager.ts:61` — new field declaration: `private serverDiscoveryPromises: Map<string, Promise<void>> = new Map();`. Good: scoped per-instance, not module-level.
- `packages/core/src/tools/mcp-client-manager.ts:148-167` — public `discoverMcpToolsForServer`:
  - guards on `serverDiscoveryPromises.get(serverName)`,
  - constructs the new promise via `Promise.resolve().then(() => internal(...))` to ensure the map insert happens *before* any synchronous part of `internal` runs (subtle but correct: prevents a re-entrancy edge case where `internal` might immediately `await` something that triggers another `discover...` for the same server),
  - cleans up in `finally` only if the map entry is still the same promise (prevents clobbering a *newer* discovery that started after this one resolved).
- `packages/core/src/tools/mcp-client-manager.ts:170-` (`discoverMcpToolsForServerInternal`) — original logic moved verbatim. The only new line here is `this.stopHealthCheck(serverName);` near the top, which prevents a health-check timer firing during the disconnect window.
- `packages/core/src/tools/mcp-client-manager.test.ts:230-318` — new test:
  - mocks `disconnect()` to hang on a manually-resolved promise so the second `discover` call is guaranteed to overlap,
  - asserts `disconnectCallsBeforeResolve === 1` (only the first call started disconnect work),
  - asserts only ONE `replacementClient` was constructed across both concurrent rediscovery calls (clean coalescing),
  - asserts a third call after the in-flight one completes performs *real* work (map cleanup verified).

## Observations
- The `if (this.serverDiscoveryPromises.get(serverName) === discoveryPromise)` check on cleanup is the right pattern — it handles the race where a fresh discovery has already replaced the entry.
- The test is unusually thorough for a coalescing fix; it covers both the "merge two in-flight" path and the "release the entry so a third call works" path. This is exactly what a future contributor would otherwise break silently.
- `stopHealthCheck(serverName)` being added to the internal function is a small but meaningful correctness fix unrelated to coalescing — worth calling out in the PR description so reviewers know it is intentional.

## Nits
- Consider adding a debug log inside the coalesce branch (`logger.debug('coalescing in-flight discovery for %s', serverName)`) — cheap observability if this misbehaves in production.
- The promise-construction pattern `Promise.resolve().then(() => internalFn(...))` works but a small comment explaining *why* the extra microtask is needed would help future maintainers.

## Verdict
`merge-as-is`
