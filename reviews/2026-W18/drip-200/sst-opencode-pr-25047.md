# sst/opencode#25047 — test: use testEffect for system prompt test

- URL: https://github.com/sst/opencode/pull/25047
- Head SHA: `d01ac73d8ce06b11ad714fb1cb279a3b7f4a3cd6`
- Size: +42 / -43 (1 file)

## Summary

Eighth entry in the running testEffect migration series (drip-194 → drip-199 covered #25053/#25052/#25051/#25050/#25049/#25048). Migrates `packages/opencode/test/session/system.test.ts` from the prior `await using tmp = await tmpdir(...)` + `Instance.provide({ directory, fn: async () => ... })` envelope onto the canonical `it.live` + `Effect.gen` shape with `tmpdirScoped({ git: true })` + `provideInstance(dir)` + `Layer.mergeAll(Agent.defaultLayer, SystemPrompt.defaultLayer, CrossSpawnSpawner.defaultLayer)`. Two `Effect.runPromise(runSkills)` calls collapse into in-generator `yield* skills` calls, asserting the same stable sort invariant.

## Observations

- `system.test.ts:23` constructs the layer via `Layer.mergeAll(Agent.defaultLayer, SystemPrompt.defaultLayer, CrossSpawnSpawner.defaultLayer)` — note the explicit `CrossSpawnSpawner.defaultLayer` addition. This is the layer set the rest of the migrated suite has been converging on; it makes the `git: true` requirement of `tmpdirScoped` actually backed by a real spawner rather than the implicit one the prior `Instance.provide` carried.
- The skill-fixture write loop at `:34-46` flattens the prior 3-iteration `for` loop into `Effect.all([...].map(([name, description]) => Effect.promise(() => Bun.write(...))), { discard: true })`. The `{ discard: true }` is the right choice here (no return values needed) but the parallelism semantics changed — the prior loop was sequential, this one fans out concurrently. For three independent `Bun.write` calls into distinct paths under a fresh `tmpdirScoped` it's safe, but flag for the next reviewer that this is now a concurrent fan-out, not a serial loop.
- The `output = first ?? (yield* Effect.fail(new NamedError.Unknown(...)))` pattern at `:71` replaces the prior implicit non-null-assertion (`first!.indexOf(...)`). Cleaner — the test now fails with a named error if the skills output is `null` rather than throwing a TypeError on `null.indexOf`. Same hardening also applied to the missing-`build`-agent path at `:64`.
- The `process.env.OPENCODE_TEST_HOME` save/restore wrapper survives the migration intact (`:53-56` save, `:78-80` restore in `finally`). The lifecycle is now interleaved with the Effect generator rather than the `Instance.provide` callback, but the try/finally on the env mutation is preserved.

## Verdict

**merge-as-is**

## Nits

- The fan-out vs. serial-loop semantics change for the skill-fixture writes is worth a one-line comment explaining `{ discard: true }` was chosen for "no return values + writes are independent."
