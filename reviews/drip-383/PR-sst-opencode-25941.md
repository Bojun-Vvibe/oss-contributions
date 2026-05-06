# sst/opencode#25941 — refactor(app): centralize sync query options

- **Head SHA**: `24ab053b15ce3ffe2d7fccda4a7df4bbf939ac1a`
- **Author**: Hona (Luke Parker)
- **Stats**: +69 / -52, 7 files

## Summary

Refactor that introduces a single `useQueryOptions()` hook on `GlobalSyncProvider` so app components stop reaching directly into `loadFooQuery(directory, sdkFor(directory))`. The hook exposes a typed map (`globalConfig`/`projects`/`providers`/`path`/`agents`/`mcp`/`lsp`/`sessions`) that internally picks global vs directory-scoped SDK via the existing `sdkCache`. Three call sites switch over (`prompt-input.tsx`, `dialog-select-mcp.tsx`, `status-popover-body.tsx`) and the standalone `mcpQueryKey`/`lspQueryKey`/`loadSessionsQueryKey` exports are folded into the inline `queryKey` literals.

## Specific citations

- New `queryOptionsApi` factory at `packages/app/src/context/global-sync.tsx:84-101` — the load-bearing change. `providers(null)` and `path(null)` correctly route to `globalSDK.client`, every other accessor routes to `sdkFor(directory)`.
- `sdkFor` hoisted from line 184 up to `:76-86` so it can be referenced before the `useQueries(() => ...)` block at `:104-107` — clean ordering fix.
- `prompt-input.tsx:1257` collapses 3-arg `loadAgentsQuery(sdk.directory, sdk.client) / loadProvidersQuery(...) / loadProvidersQuery(...)` to `queryOptions.agents(sdk.directory) / queryOptions.providers(null) / queryOptions.providers(sdk.directory)` — kills the brittle `useGlobalSDK()` import at `:19`.
- `child-store.ts:8` swaps `getSdk: (dir) => OpencodeClient` for `queryOptions: typeof queryOptionsApi` (richer abstraction surface).
- `global-sync.tsx:248` `loadSessionsQueryKey(key)` → `...queryOptionsApi.sessions(key)` spread — semantically equivalent (key shape `[directory, "loadSessions"]` preserved at `:101`).

## Verdict

**merge-after-nits**

## Rationale

The refactor is correct and reduces duplication, but there are minor smells. (1) `sessions` accessor at `:101` returns only `{ queryKey }` — no `queryFn`, asymmetric with the other accessors that return full `queryOptions(...)`; an inline comment naming the rationale (sessions still have a custom fetch wrapper at `:246-262`) would help. (2) Removing the named `mcpQueryKey`/`lspQueryKey`/`loadSessionsQueryKey` exports is mildly risky — any out-of-tree consumer (extensions, downstream forks) breaks silently; a single `@deprecated` re-export for one minor would soften the landing. (3) PR body claims "key-only consumers route through `queryOptions.keys`" but the new accessor returns `.queryKey` (singular) — naming consistency nit. No test coverage for the new `queryOptionsApi` shape itself, but the existing `child-store.test.ts:23-27` patch passes a stub, which proves the consumer-side contract holds.

