# block/goose #8873 — perf: deduplicate _goose/providers/list RPC call at startup

- PR: https://github.com/block/goose/pull/8873
- Head SHA: `958c7989eba77724fdb57739870c930ca720bc3b`
- Files changed: 3 (`+60/-35`) — `ui/goose2/src/app/hooks/useAppStartup.ts`, `ui/goose2/src/shared/api/acp.ts`, `ui/goose2/src/shared/api/acpApi.ts`

## Verdict: merge-as-is

## Rationale

- **Right shape for the perf optimization.** Before: `useAppStartup` ran `loadProviders` (which called `directAcp.listProviders()` → RPC `_goose/providers/list`) and `loadProviderInventory` (which called `getProviderInventory()` → the same RPC under the hood) in parallel via `Promise.allSettled` at `useAppStartup.ts:147-152`. After: a single `loadProvidersAndInventory` at `useAppStartup.ts:48-72` runs the inventory fetch once and *derives* the provider list from the same response via the new `discoverAcpProvidersFromEntries` helper at `acp.ts:46-51`. One RPC instead of two on every cold start — exactly the dedup the title promises.
- **Helper extraction is the right structural choice.** `acpApi.ts:43-57` extracts `buildProviderListFromEntries(entries)` from the inline body of `listProviders`. Both consumers now call the same shape: `listProviders` (which still does `client.goose.GooseProvidersList({...})` then passes `result.entries` through the helper at `:65`) and the new `discoverAcpProvidersFromEntries` (which receives entries already-fetched by inventory). The `DEPRECATED_PROVIDER_IDS` filter and the `DEFAULT_PROVIDER` prepend now live in exactly one place — kills the drift class where one path filters deprecated IDs and the other doesn't.
- **`resolveProvidersCatalog` private function** at `acp.ts:53` factors out the seen-set dedup that was previously inline in `discoverAcpProviders`. Both `discoverAcpProviders` and `discoverAcpProvidersFromEntries` route through it (lines `:42` and `:50` respectively) — same dedup semantics across the two entry points, no risk of subtle divergence.
- **Loading-state lifecycle is correctly merged.** `useAppStartup.ts:50-51` sets *both* `store.setProvidersLoading(true)` and `inventoryStore.setLoading(true)` before the await, and the `finally` at `:64-66` clears both flags symmetrically. Prior code set them in two separate `try/finally` blocks; the merge could have lost a `setLoading(false)` if not careful — the new code gets it right.
- **Error message updated to match the merged operation.** `:60-63` now says "Failed to load providers and inventory on startup" instead of the prior "Failed to load provider inventory" — the merged function loads both, and the error message reflects that. Same for the `perfLog` at `:54-56` which now reports `entries=${entries.length}, providers=${providers.length}` so the perf telemetry covers both quantities.
- **Promise.allSettled call shape preserved.** `:144-149` swaps `loadProviders(), inventoryLoad` for `providersAndInventoryLoad` — the array shrinks by one but `loadPersonas`, `loadSessionState` are untouched. The downstream `void inventoryLoad.then(...)` at `:150-152` is renamed `void providersAndInventoryLoad.then(...)` and still gets the entries — the `.then((entries) => refreshConfiguredProviderInventory(entries))` chain is preserved verbatim, so the post-load refresh logic still fires.
- **`DEPRECATED_PROVIDER_IDS` and `DEFAULT_PROVIDER` exported.** `acpApi.ts:30-39` widens these from `const` to `export const` — necessary because the new `buildProviderListFromEntries` is exported and consumers might want to introspect either constant. No risk of accidental mutation since both are `const` (and `Set` is referenced not constructed in callers). Clean widening.

## Why merge-as-is

This is a straightforward dedup with three correct structural moves: (1) merge the two parallel RPCs into one, (2) extract the shared filter+prepend logic, (3) preserve dedup semantics across both entry points via a private helper. The diff is internally consistent, the loading-state lifecycle is correctly merged, the error/perf telemetry reflects the new operation, and existing tests should pass unchanged because `listProviders`'s public contract is preserved (it still returns `[DEFAULT_PROVIDER, ...filtered_entries]`). PR body lists `pnpm test` + biome + build + manual verification — adequate for a UI-layer perf change with no behavior change in the public API.

## Optional nits (non-blocking)

- The `Array<{ providerId: string; providerName: string }>` parameter type appears at `acp.ts:48` and `acpApi.ts:46` with the exact same shape literal. A shared `ProviderInventoryEntry` type or `Pick<>` from the inventory module's existing entry type would be cleaner — but both files already import from `acpApi`, so it's a one-line refactor for a future PR.
- No regression test for the dedup itself — i.e. "startup makes exactly one `_goose/providers/list` RPC, not two." A spy on `client.goose.GooseProvidersList` asserting `toHaveBeenCalledTimes(1)` would lock the perf invariant. Optional because the structural change makes the regression hard to reintroduce without noticing.
