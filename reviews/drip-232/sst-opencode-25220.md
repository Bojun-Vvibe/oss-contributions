# sst/opencode#25220 — fix(desktop): Model selection UI becoming blank

- **Author:** Eric Guo (Eric-Guo)
- **Head SHA:** `304c979cfd8d0115b7c8eed29ea24adb74c9f4fe`
- **Base:** `dev`
- **Size:** +8 / -2 across 2 files
- **Files changed:** `packages/app/src/context/global-sync/child-store.ts`, `packages/app/src/hooks/use-providers.ts`

## Summary

Closes #25222 — desktop model-selection UI rendering blank — caused by regression from #25077 (the recent child-store refactor). Two-axis fix: (a) the Solid-JS child-store wasn't ever forwarding fresh `providerQuery.data` into the per-project store after initial creation, so a project's `provider` slot stayed at its constructor-time empty value forever; (b) the `useProviders()` hook was gating on a `provider_ready` flag that v2 never sets, so even when provider data arrived nothing in the UI knew to switch from the global fallback to the project-scoped list.

## Specific code references

- `child-store.ts:1`: import widened to add `createEffect` from `solid-js` — straightforward Solid-JS reactive primitive needed for the new wiring.
- `child-store.ts:229-234`: the load-bearing fix. New `createEffect(() => { const data = providerQuery.data; if (!data) return; child[1]("provider", data) })` block inside `createChildStoreManager`'s child-creation path. This is the contract-level missing piece — `providerQuery` was being constructed but its data was never re-piped into the child store on subsequent settled responses. The `if (!data) return` guard correctly skips the pre-resolution `undefined` state so the child's `provider` slot isn't clobbered to empty during the in-flight window. Setup runs *after* `disposers.set(key, dispose)` so the effect is registered under the same disposer that the `runWithOwner`/`onCleanup` chain already manages — proper Solid-JS cleanup hygiene.
- `use-providers.ts:25`: predicate flipped from `if (projectStore.provider_ready) return projectStore.provider` to `if (projectStore.provider.all.length > 0) return projectStore.provider`. This is the correct shape because v2 `child-store` no longer sets a separate `provider_ready` boolean — it sets the `provider` field directly with the full `{all, default}` shape. Gating on `.all.length > 0` queries the actual data structure rather than a phantom flag.

## Reasoning

The root cause is a real Solid-JS reactive-graph bug — `providerQuery.data` is a reactive accessor, but it was only being read once at child-creation time, not subscribed to. The `createEffect` correctly creates the subscription. Two notes worth maintainer attention:

1. The `child[1]("provider", data)` setter call replaces the *entire* `provider` slot wholesale on every settled response. If `provider` is shaped `{all: [...], default: ...}` and any consumer holds a reference to the old `.default`, that reference now points at stale data. The replacement is correct for the bug at hand (the slot was never updated, now it is) but if downstream selectors use `provider.default` directly they should be re-checked for staleness. A `produce(...)`-based merge would be more conservative.
2. The `use-providers.ts` change drops the `provider_ready` predicate entirely. If `provider.all = []` is ever a *valid* state for "loaded, but this project has no providers configured" (vs the in-flight pre-load state), this fix would incorrectly fall through to the global default in that case. A `provider_loaded` boolean alongside the data array would distinguish these — worth confirming the empty-providers case is impossible by construction or that falling through to global is actually the desired UX.

The PR body's "It works in my machine" verification line is light — the real check is whether the regression test for #25077 caught any of this (it didn't — the bug shipped). A targeted unit test for `createChildStoreManager` asserting that `child.provider` updates on subsequent `providerQuery.data` settles would prevent recurrence.

## Verdict

**merge-after-nits**
