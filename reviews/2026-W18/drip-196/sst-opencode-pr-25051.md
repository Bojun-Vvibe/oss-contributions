# sst/opencode PR #25051 — test: use Effect test helper for agent colors

- PR: https://github.com/sst/opencode/pull/25051
- Head SHA: `d1953fc8999fef3464222337edf78121619fa0a2`
- Files touched: `packages/opencode/test/config/agent-color.test.ts` (+43/-51, 1 file)

## Specific citations

- Net is -8 LOC and the migration shape exactly mirrors drip-194 #25053 and drip-195 #25052: replace `await using tmp = await tmpdir()` + `Instance.provide({...})` + `AppRuntime.runPromise(Config.Service.use(...))` with `it.live(...)` over `Effect.gen` that yields `tmpdirScoped()` and pipes through `provideInstance(dir)`. The composed test layer at `:14` is `Layer.mergeAll(AgentSvc.defaultLayer, CrossSpawnSpawner.defaultLayer)` — `CrossSpawnSpawner.defaultLayer` is added because the `AgentSvc` factory transitively spawns processes during the `agent.get` path that the second test exercises.
- `writeConfig` helper at `:18-27` lifts the inline JSON-write into a small `Effect.promise`-wrapped `Bun.write`. The two callers at `:30-33` and `:46-49` differ only by their `agent` payload, so the helper is justified — and unlike a plain async function it returns an Effect that composes with the surrounding `Effect.gen`.
- The first test still pierces the layer to call `AppRuntime.runPromise(Config.Service.use(...))` at `:36` rather than going through the test layer's services. This is fine for a config-parse smoke test (the goal is to assert `config.toml` parses), but it means this test isn't actually exercising the services injected by `it.live`'s composed layer — only the second test does.
- The third `test("Color.hexToAnsiBold ...")` at `:55-58` (in original) is left untouched as a plain `bun:test` test — correct, since it tests a pure function and adding Effect ceremony would be cargo-culting.

## Verdict: merge-as-is

## Rationale

Mechanical, narrow, and matches the established migration shape across the testEffect series (drip-194 #25053, drip-195 #25052, and now this). Net -8 LOC. No new behavior, no new code paths, no test parallelization concerns since the only stateful piece is a per-test `tmpdirScoped`. The pure `Color.hexToAnsiBold` test correctly stays as plain `test()`. One mild observation worth a follow-up but not a blocker: the first test's `AppRuntime.runPromise(Config.Service.use(...))` bypasses the `it.live`-composed layer; if the broader migration intent is to have all config tests resolve services through the test layer (so `CrossSpawnSpawner.defaultLayer` and friends are uniformly stub-able), a future pass could route this through `Config.Service.use` from inside the `Effect.gen`. As-is, it's still strictly better than the pre-PR shape.
