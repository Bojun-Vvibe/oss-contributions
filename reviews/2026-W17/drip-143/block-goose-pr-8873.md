# block/goose#8873 — perf: deduplicate `_goose/providers/list` RPC call at startup

- Head SHA: `958c798`
- Author: matt2e
- Files: 3 / +60 / −35

## Summary

Eliminates a duplicate `_goose/providers/list` RPC at goose2 UI startup by merging the previously-independent `loadProviders()` and `loadProviderInventory()` startup calls into a single `loadProvidersAndInventory()`. The provider inventory RPC already returns the entries needed to derive the ACP provider list, so the second `discoverAcpProviders()` call (which internally fetches the same data) was redundant. Extracts a shared `buildProviderListFromEntries()` helper so both the startup path (via new `discoverAcpProvidersFromEntries()`) and the on-demand `listProviders()` path use identical dedup + DEFAULT_PROVIDER-prefix logic.

## Specific observations

- Load-bearing extraction at `ui/goose2/src/shared/api/acpApi.ts:46-58` — `buildProviderListFromEntries(entries)` returns `[DEFAULT_PROVIDER, ...entries.filter(!DEPRECATED).map(...)]`. Single source of truth for the provider-list shape. Both `listProviders()` at `:65` (the RPC-fetching path) and the new `discoverAcpProvidersFromEntries()` at `:50` of `acp.ts` flow through it, so a future change to "what counts as a deprecated provider" or "what fields go on the provider tuple" lands in one place. Right shape.
- `DEPRECATED_PROVIDER_IDS` and `DEFAULT_PROVIDER` promoted from `const` to `export const` at `acpApi.ts:30/35` — necessary for the cross-module helper. The `DEPRECATED_PROVIDER_IDS` set contains `["claude-code", "codex", "gemini-cli"]` which is unchanged from before; visibility-widening only.
- `acp.ts:41-50` — new `discoverAcpProvidersFromEntries(entries)` derives ACP providers from already-fetched inventory entries via `directAcp.buildProviderListFromEntries(entries)` then runs the existing `resolveProvidersCatalog()` dedup pass (factored out at `:54-68` from the previous inline logic in `discoverAcpProviders()`). The `discoverAcpProviders()` path is preserved for callers that need the standalone fetch — no breaking change to the public surface.
- Startup orchestration at `ui/goose2/src/app/hooks/useAppStartup.ts:51-83` — the new `loadProvidersAndInventory` sets BOTH `store.setProvidersLoading(true)` AND `inventoryStore.setLoading(true)` before the single `getProviderInventory()` await, then on success calls BOTH `inventoryStore.setEntries(entries)` AND `store.setProviders(discoverAcpProvidersFromEntries(entries))`. The `finally` block clears both loading flags. This is the load-bearing atomicity — both stores transition from loading→loaded together, so a UI subscriber observing one flag won't see a stale value for the other.
- `Promise.allSettled` at `:148-152` was previously `[loadPersonas(), loadProviders(), inventoryLoad, loadSessionState()]` (4 entries) and is now `[loadPersonas(), providersAndInventoryLoad, loadSessionState()]` (3 entries). One round-trip eliminated, one less concurrent connection during startup.
- Error semantics — the previous code had two independent `try/catch` blocks logging `"Failed to load ACP providers on startup"` and `"Failed to load provider inventory on startup"` separately. The combined version logs once: `"Failed to load providers and inventory on startup"`. A single failure now masks which of the two stores has stale data — but since both fields come from the *same* RPC call now, "one failed and the other succeeded" is no longer a possible state, so the merged error message is correct.
- Perf-log message at `:79` correctly reports both counts: `entries=${entries.length}, providers=${providers.length}` — preserves observability that was previously split across two `[perf:startup]` lines.
- No test diff visible — `pnpm test` mentioned in test plan but no specific test cell pinning "startup makes exactly one `_goose/providers/list` RPC call". A spy-and-assert-once test at the integration boundary would prevent regression-by-future-refactor.

## Verdict

`merge-as-is`

## Rationale

Surgical, well-scoped perf fix that removes a duplicate RPC by recognizing that two startup paths were both fetching the same `_goose/providers/list` data. The shared-helper extraction is the right engineering pattern (single source of truth for provider-list shape), the atomicity of paired-store updates is preserved, error semantics correctly fold from two independent failures into one (since the underlying call is now also one), and the perf-log line preserves observability. The lack of an "exactly-one RPC" test is the only thing I'd flag for follow-up, but it's not material — the refactor structurally guarantees the property by deleting the second call site, and a CI assertion would have to wrap the network layer to add real value. Ship it.

