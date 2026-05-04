# Review: sst/opencode #25694

- **Title:** fix: propagate InstanceRef into ScopedCache lookup in InstanceState.get()
- **Head SHA:** `949e848d8cb5d0d65a9c1a2e6ca7b3ef210d6619`
- **Scope:** +61 / -1 across 2 files
- **Drip:** drip-338

## What changed

`InstanceState.get` previously called `ScopedCache.get(self.cache, yield*
directory)` directly. When the cache lookup itself runs outside of
AsyncLocalStorage (e.g. inside a `Effect.fn` boundary that captures the
calling fiber's context but loses ALS), the cache initializer fired without
an `InstanceRef` in scope and observed the wrong directory. This patch
captures `ctx = yield* context` from the calling fiber, then passes it both
as the cache key (`ctx.directory`) and as a provided service
(`Effect.provideService(InstanceRef, ctx)`) so initializers see a stable
ref regardless of the ALS state at lookup time.

## Specific observations

- `packages/opencode/src/effect/instance-state.ts:60-66` — the new pattern
  is correct: pull `ctx` once, use `ctx.directory` as the cache key, and
  re-provide `InstanceRef` to the lookup pipe. This guarantees parity
  between key and initializer view of the instance.
- `packages/opencode/src/effect/instance-state.ts:60` — `context` import is
  not visible in the diff hunk; assume it's already imported (the file
  already references `directory` and `InstanceRef`). Worth a quick local
  check that `context` is the existing helper in this module rather than a
  new symbol that should also be added to imports.
- `packages/opencode/test/effect/instance-state.test.ts:432-491` — new test
  `InstanceState.get resolves InstanceRef from calling fiber even when
  ScopedCache lookup has no ALS` exercises the regression precisely:
  - Two separate `Instance.provide` calls with different `directory` paths
    (`one.path`, `two.path`) confirm the cache keys differ.
  - `initCalls` accounting confirms no spurious re-init.
  - Third `Instance.provide` for `one.path` confirms the cached value is
    reused (`initCalls === 2`).
- The test wires through `Context.Service` + `ManagedRuntime` which is a
  realistic shape for service-bound state in this codebase, not a contrived
  unit harness. Good signal.

## Risks

- Pulling `ctx` eagerly out of the calling fiber means any later mutation
  to the fiber's `InstanceRef` won't be reflected for the cache lookup.
  That's already the contract for ScopedCache (key is captured at lookup
  time), so this is intentional rather than a regression.
- Performance: one extra `yield* context` per `get`. Negligible.

## Verdict

**merge-as-is** — targeted bug fix with a regression test that names
exactly the failure mode it prevents.
