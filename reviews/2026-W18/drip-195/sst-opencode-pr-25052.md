# sst/opencode PR #25052 — test: use testEffect for plugin workspace adaptor

- PR: https://github.com/sst/opencode/pull/25052
- Head SHA: `d1518d6ec26bf9d6a63f1eea46b650b1df9925fa`
- Files touched: `packages/opencode/test/plugin/workspace-adaptor.test.ts` (+66/-65, 1 file)

## Specific citations

- Imports flip at `workspace-adaptor.test.ts:1-8`: drops `test` and `tmpdir`; adds `Layer`, `CrossSpawnSpawner`, `provideTmpdirInstance`, `testEffect` — same shape as the sibling `trigger.test.ts` migration in PR #25053 from drip-194.
- New `it = testEffect(Layer.mergeAll(Plugin.defaultLayer, CrossSpawnSpawner.defaultLayer))` at `:16` — replaces the per-test `Effect.runPromise(...).pipe(Effect.provide(Plugin.defaultLayer))` boilerplate.
- The test body at `:36-110` is converted from `await using tmp = await tmpdir({ init: ... })` plus an awaited `Instance.provide({ fn: async () => Effect.gen(...) })` into a single `it.live("...", () => provideTmpdirInstance((dir) => Effect.gen(function* () { ... })))` — the `Effect.gen` block now `yield* Effect.promise(() => Bun.write(...))` for both the `plugin.ts` and `opencode.json` writes (no parallelization here, unlike #25053's `Effect.all([..., ...], { discard: true, concurrency: 2 })` shape).
- The pre-migration shape used `await tmp.path` then `Instance.provide({ directory: tmp.path, fn: ... })` — the new shape lets `provideTmpdirInstance` own both the tmpdir and the `Instance` scope, presumably tying disposal to the Effect scope.
- `afterAll` at `:31-33` is preserved (resets `OPENCODE_DISABLE_DEFAULT_PLUGINS`); `afterEach` is gone since `Instance.disposeAll()` is no longer needed under the scoped runner.

## Verdict: merge-as-is

## Concerns / nits

1. **Inconsistency with sibling #25053**: that PR's two writes use `Effect.all([..., ...], { discard: true, concurrency: 2 })` — this PR's two writes are sequential `yield* Effect.promise(...)` calls at `:39-67` and `:69-83`. They're independent (different file paths, no ordering dependency); applying the same `Effect.all` parallelization here would match the pattern and shave a tiny amount of test wall-time. Not a blocker — net diff is already +1 LOC and stylistic — but worth doing as a follow-up so the two trigger/workspace migrations are pattern-twins.
2. **`it.live` rationale**: same nit as #25053 — drop a one-line comment naming why `live` (real `Bun.write` IO and real plugin loader) so the next refactor doesn't "simplify" it back to `it.effect` and silently break the fixture writes.
3. **Net +1 LOC** for an equivalent-coverage migration is fine; the win is not LOC, it's the lifecycle being scoped to the Effect rather than imperative `afterEach`/`disposeAll` ceremony. Same migration shape as the in-flight test sweep across `instruction.test.ts`, `system-prompt.test.ts`, `runner-deadlock.test.ts`, etc. (PRs #25045-#25051) — pattern is clearly load-bearing for the Effect-test consolidation.
4. The `process.env.OPENCODE_DISABLE_DEFAULT_PLUGINS = "1"` write at module top-level (line `:11`) is module-scoped and racy with parallel test files that touch the same env var; this isn't a regression introduced by the migration, but the migration is a good moment to flip it to a layer-provided `Flag` the same way #25035 retired global flag reads in HttpApi auth middleware. Worth a follow-up.
