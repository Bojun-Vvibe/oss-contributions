# sst/opencode#25277 — Move instance loading into Effect service

- **PR**: https://github.com/sst/opencode/pull/25277
- **Head SHA**: `7b58be118bad69560171f50d3b57944ffba3480a`
- **Verdict**: `merge-after-nits`

## What it does

Promotes the `Instance.*` module-level singleton (a hand-rolled cache + four
loose async functions closing over it) into a proper Effect `Context.Service`
named `InstanceStore`, then rewires the legacy `Instance` facade and the HttpApi
instance-context middleware to call through the service. +300 / -153.

## What's load-bearing

- `packages/opencode/src/project/instance-store.ts:1-159` — the new service.
  Two state fields (`cache: Map<string, Promise<InstanceContext>>` and
  `disposal.all: Promise<void> | undefined`) are now layer-scoped, not
  module-scoped; that's the actual win, since each `Layer.effect` build now
  produces an isolated cache (testable, swappable, finalizer-bound).
- `packages/opencode/src/project/instance-store.ts:148` —
  `yield* Effect.addFinalizer(() => disposeAll().pipe(Effect.ignore))`
  wires layer scope to dispose-all, which means HttpApi reload paths that build
  scoped layers can no longer leak instances on layer dispose. This is the
  shape that closes the structural leak the prior module-let couldn't.
- `packages/opencode/src/project/instance.ts:1-69` — the public `Instance`
  facade is preserved as a thin proxy: `Instance.load`, `Instance.reload`,
  `Instance.dispose`, `Instance.disposeAll` all delegate to
  `InstanceStore.runtime.runPromise(...)`. Backwards-compatible at the call
  sites, which is the right discipline for a refactor of this scope.
- `packages/opencode/src/server/routes/instance/httpapi/middleware/instance-context.ts`
  changes the middleware to consume `InstanceStore.Service` from the request
  layer rather than entering legacy `AsyncLocalStorage`, which is the real
  reason the PR exists per the summary ("routed requests receive `InstanceRef`
  and `WorkspaceRef` without entering legacy ALS").
- `packages/opencode/src/server/routes/instance/httpapi/lifecycle.ts:7,21,31,53` —
  `MarkedInstance` now carries `store: InstanceStore.Interface`, and
  `restoreMarked` flips signature from `(marked, fn: () => A)` to
  `(marked, effect: Effect.Effect<A>)`. The bridge that "still publishes
  events through legacy ALS helpers" is honest about being a bridge, not a
  permanent API.
- `packages/opencode/test/project/instance.test.ts` — 85 new test lines
  covering the service in isolation, structurally the right place to pin the
  layer-scoped cache contract.

## Concerns / nits

1. **`Effect.runPromise(boot(...))` inside `Effect.fn` is a smell.** At
   `instance-store.ts:50` the load path does
   `Effect.promise(() => track(directory, Effect.runPromise(boot({...}))))`.
   Wrapping `runPromise` in `Effect.promise` discards the parent fiber's
   interruption signal — if the request layer is interrupted while `boot` is
   in flight, the boot continues to completion. Same shape at
   `:62` (reload). Prefer `Effect.scoped` or yielding the boot effect
   directly so interruption propagates.
2. **`disposal.all` finalizer race.** `:148` uses `Effect.ignore` on the
   layer finalizer so disposeAll errors are swallowed. The only signal
   anyone gets is the `Log.Default.warn` inside the loop. A failed dispose
   in production becomes silent; please at minimum emit a structured event
   (`server.instance.dispose.failed`) on the GlobalBus so operators can
   alert.
3. **`Service.of({...})` at `:152` doesn't name the methods at the type
   level.** Consider `Effect.fn("InstanceStore.method")` consistently — most
   are named already (good), but `Service.of` won't surface a missing
   trace-name regression.
4. **Module-level `runtime = makeRuntime(Service, defaultLayer)` at `:158`
   re-introduces a module singleton.** The whole point of the refactor is
   that the cache is layer-scoped, but `Instance.load`'s implementation
   reaches for `InstanceStore.runtime.runPromise(...)` (per
   `instance.ts:9-11`), which materializes a single shared runtime per
   process. Acceptable as a transition shim; please leave a `// TODO:
   thread runtime through callers and remove module-level runtime` comment
   so future readers know this is the load-bearing follow-up.
5. **Test gap: cache identity under concurrent loads.** The
   `track(directory, ...)` pattern at `:32-39` deletes the cache entry on
   error only if it's still the current task. There's no test for
   "two concurrent `load` calls for the same directory share the same
   promise and observe the same `InstanceContext`." Cheap to add and pins
   the most important behavior of the cache.

## Verdict

`merge-after-nits` — the refactor is clean and the layer-scoping is the
right structural fix. The interruption-propagation concern (#1) and the
module-level `runtime` (#4) are both shaped like "follow-up PR" rather than
"block this one." Nits #2, #3, #5 are pre-merge polish.
