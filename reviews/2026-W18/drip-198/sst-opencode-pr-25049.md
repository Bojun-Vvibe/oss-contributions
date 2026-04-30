# sst/opencode PR #25049 — test: use Effect test helper for app runtime logger

- PR: https://github.com/sst/opencode/pull/25049
- Head SHA: `91c3db703fe71a44d743105853947f7b569205de`
- Files touched: 1 (`packages/opencode/test/effect/app-runtime-logger.test.ts` +51/-45). Net +6 LOC.

## Specific citations

- The migration shape is established at `packages/opencode/test/effect/app-runtime-logger.test.ts:9` — `const it = testEffect(CrossSpawnSpawner.defaultLayer)` — which mirrors the pattern already landed in drip-194 #25053 (plugin triggers), drip-195 #25052 (workspace adaptor), and drip-196 #25050 (retry policy). All four `test(...)` blocks at `:23-95` are converted to `it.live(...)` callbacks returning `Effect.gen` generators, so they now run inside the composed Effect runtime rather than as bare `async` arrow functions.
- The two scope-bound rewrites at `:60-67` (`AppRuntime attaches InstanceRef from ALS`) and `:78-95` (`EffectBridge preserves logger and instance context across async boundaries`) replace the prior `await using tmp = await tmpdir({ git: true })` + `Instance.provide({ directory: tmp.path, fn: ... })` envelope with `tmpdirScoped({ git: true })` from `../fixture/fixture` (yielded into the Effect's scope), composed with `provideInstance(dir)` on the outer `Effect.promise(...)`. That moves cleanup off the JS `using` disposal protocol and onto Effect's scope-finalizer chain, which is the actual point of the migration series.
- `AppRuntime.runPromise(...)` and `makeRuntime(...).runPromise(...)` are still wrapped in `Effect.promise(() => ...)` at `:35` / `:50` / `:64` / `:84` — the runtime calls remain Promise-returning, the test scaffolding around them is what moves. So the SUT (`AppRuntime` + `EffectBridge.make` + `InstanceRef`) is unchanged, only the test composition changes.

## Verdict: merge-after-nits

## Rationale

This is the fifth file in the testEffect migration series and follows the established shape exactly — `it.live` over `Effect.gen`, `tmpdirScoped` + `provideInstance` over the `await using` + `Instance.provide` envelope, `CrossSpawnSpawner.defaultLayer` as the layer arg. The +6 LOC delta and the minimal-surface mechanical character are both consistent with the prior series PRs, which is exactly the right shape: any deviation here would be more interesting than the change itself. The hidden behavior change relative to the prior series is small but real — this file's tests legitimately need `CrossSpawnSpawner.defaultLayer` (because `tmpdir({ git: true })` shells out to `git init`) where #25051 (agent colors) does not, so the layer choice is load-bearing rather than ceremonial. Two nits: (1) the inner `await using tmp` cleanup contract is gone but `tmpdirScoped` now owns the equivalent finalizer through the Effect scope — worth a one-line comment at `:60` so future readers see the disposal contract didn't disappear, it moved; (2) all four `runPromise` call sites still synthesize a fresh `AppRuntime` per test, so the migration leaves on the table the cross-test runtime sharing that `it.live` would otherwise enable, and a follow-up PR collapsing those to a layer-scoped fixture would be the obvious next step in the series. Neither concern blocks merge — the file lands at parity with the established series.
