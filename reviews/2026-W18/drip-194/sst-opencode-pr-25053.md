# sst/opencode PR #25053 — test: use testEffect for plugin triggers

- PR: https://github.com/sst/opencode/pull/25053
- Head SHA: `f98478e6c1d1a21f931a3eb68932d688f7a2993d`
- Files touched: `packages/opencode/test/plugin/trigger.test.ts` (+64/-78)

## Specific citations

- Imports flip at `trigger.test.ts:1-8`: drops `afterEach`, `test`, and `tmpdir` (the imperative fixture); adds `Layer`, `CrossSpawnSpawner`, `ModelID`, `ProviderID`, `provideTmpdirInstance`, and `testEffect`.
- Test runner setup at `:14`: `const it = testEffect(Layer.mergeAll(Plugin.defaultLayer, CrossSpawnSpawner.defaultLayer))` — the new canonical Effect-test entrypoint, replacing the previous per-test `Effect.runPromise(...).pipe(Effect.provide(Plugin.defaultLayer))` boilerplate at the old `:46-65`.
- Per-test `Instance.disposeAll()` `afterEach` block at the old `:13-15` is removed; lifecycle now flows through `provideTmpdirInstance` which presumably scopes the Instance per Effect.
- `withProject` helper at `:25-50` now returns an `Effect` instead of an awaited resource — the file-creation pair is wrapped in `Effect.all([..., ...], { discard: true, concurrency: 2 })` at `:28-46` (parallel `Bun.write` for `plugin.ts` and `opencode.json`).
- Shared assertion harness `triggerSystemTransform` at `:52-68` extracted as an `Effect.fn` with the trace name `"PluginTriggerTest.triggerSystemTransform"` — `ProviderID.anthropic` and `ModelID.make("claude-sonnet-4-6")` replace the previous `as any` cast at the old `:55-58`.
- Two `it.live(...)` cases at `:71-87` and `:89-104` cover the sync and async hook variants respectively.
- `systemHook` constant at `:15` (`"experimental.chat.system.transform"`) deduplicates the previously-string-duplicated hook key.

## Verdict: merge-as-is

## Concerns / nits

1. **`it.live` choice over `it.scoped` / `it.effect`** — the test interacts with real filesystem via `Bun.write` and presumably real plugin-loader IO so `live` is the right call; consider a one-line comment naming why `live` (vs the Effect TestClock variants) so the next refactor doesn't "simplify" it back to a non-live runner and silently break filesystem fixtures.
2. **`concurrency: 2` at `:46`** — both writes are independent so this is a free perf win, but document `discard: true` is intentional (we don't need the write results, only the side effect) for the next reader.
3. **The `as any` removal at `:60`** is a real type-safety upgrade — switching to `ProviderID.anthropic` + `ModelID.make("claude-sonnet-4-6")` exercises the schema constructors which would catch a future enum/brand drift at compile time. Same level of test coverage but stronger contract.
4. Net **-14 lines** for two equivalent tests with stronger typing and a parallel write — exemplary migration shape.
