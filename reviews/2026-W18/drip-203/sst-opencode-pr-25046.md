# sst/opencode #25046 — test: use testEffect for instruction tests

- **Author:** kitlangton (Kit Langton)
- **SHA:** `0bcdec4`
- **State:** OPEN
- **Size:** +226 / -312 across 1 file
  (`packages/opencode/test/session/instruction.test.ts`)
- **Verdict:** `merge-as-is`

## Summary

Tenth entry in the running testEffect migration series (drip-194 → drip-203),
porting `Instruction.resolve` coverage off the bare
`Effect.runPromise(...).pipe(Effect.provide(Instruction.defaultLayer))` shim
+ `await using tmp = await tmpdir({ init: ... })` pattern onto
`testEffect(Layer.mergeAll(Instruction.defaultLayer, CrossSpawnSpawner.defaultLayer))`
(`instruction.test.ts:9`) + new helpers `tmpWithFiles` and `withFiles` that
combine `tmpdirScoped()`/`provideTmpdirInstance(...)` with a parallel
`writeFiles(dir, files)` (`Effect.all(..., { discard: true })`).

## Reasoning

Series-conformant on every axis:

1. **Layer composition is the right shape.** `CrossSpawnSpawner.defaultLayer`
   is now part of the test runtime because `Instruction.resolve` walks
   directories with the spawn-based git ignore detection — without that
   layer the test would have silently fallen through to a missing-service
   error. The `Layer.mergeAll(...)` form matches drip-202's
   `runner.test.ts:14` shape exactly.

2. **All `await using tmp = await tmpdir({ init })` blocks are gone.** The
   replacement `withFiles({...}, (dir) => Effect.gen(function* () { ... }))`
   keeps the file-seeding semantics inside the Effect runtime so a setup
   failure now produces a typed `Effect` error rather than a thrown
   `Promise`-domain rejection that's hard to thread back into the test
   reporter. This is the same correctness win that drip-201's
   `system-prompt.test.ts` migration captured.

3. **Test bodies preserve the `Instruction.resolve(...)` assertion shape
   byte-for-byte.** Each test still passes the same `loaded(filepath)`
   message fixture and asserts on the same returned-paths set; the only
   change is the runtime envelope.

4. **`provideTmpdirInstance` is the correct service-injection helper for
   tests that need both `Instance` *and* a fresh tmpdir.** Previously the
   test reached for the bespoke `Instance.use(dir, async () => ...)` pattern
   which required nested async closures; the new helper inverts the
   relationship and lets the test body stay flat in `Effect.gen`.

Migration is mechanical, no behavioral diff in what's being tested. Mergeable
as-is. The series can keep going — the next obvious candidate is whatever
`test/session/*.test.ts` file still imports `await using tmp = await
tmpdir({...})` directly (`grep -l "tmpdir({" packages/opencode/test/`).
