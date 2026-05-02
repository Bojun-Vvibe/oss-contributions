# sst/opencode PR #25377 — Migrate test inits from Promise to Effect

- **URL**: https://github.com/anomalyco/opencode/pull/25377
- **Head SHA**: `847c408c07e34b44f178e99a9885d8febf8cf8e7`
- **Files touched**: `packages/opencode/src/project/instance.ts`, plus broad test sweeps under `packages/opencode/test/provider/*.test.ts`

## Summary

Switches `Instance.provide({ init })` and `Instance.load` from accepting a `() => Promise<unknown>` to an `Effect.Effect<void>`. Tests are mechanically migrated to wrap their previous async closures in `Effect.promise(async () => {...}).pipe(Effect.asVoid)`. Production code-path uses `Effect.callback` to bridge ALS context provision through the runtime.

## Comments

- `instance.ts:30-37` — the new `Effect.callback<void>((resume) => { context.provide(ctx, () => { Effect.runPromise(init).then(...) }) })` recreates a promise-on-effect-on-promise sandwich. If `init` is already an `Effect`, calling `Effect.runPromise(init)` inside an `Effect.callback` from inside an outer `Effect.gen` schedules the init on the *default* runtime, not the InstanceStore runtime. Verify that's intentional — otherwise we lose any layers the store provided beyond `InstanceRef`.
- `instance.ts:34` — `(err) => resume(Effect.die(err))` turns init failures into defects (unrecoverable). Old code did `Effect.promise(() => init())` which similarly elevates rejections to defects, so behaviour is preserved, but now there's no opportunity to convert a typed init failure into a typed `Effect` error. Worth a `try`/`catch`-style overload for callers that want recoverable init.
- `amazon-bedrock.test.ts:58-63` (and ~7 identical sites) — the `Effect.promise(async () => { set("AWS_REGION", "us-east-1"); set("AWS_PROFILE", "default") }).pipe(Effect.asVoid)` boilerplate gets repeated verbatim 8+ times. A small test helper `envInit({...})` would make these read better and make the next migration step (real `Effect.sync`) trivial.
- The migration is purely mechanical — no test was changed to actually *use* effect features (resource-scoped env, Layer-based config). The PR title says "migrate" but realistically this is a type-shape change; consider renaming to "refactor: type Instance.provide init as Effect" so the next reviewer doesn't expect deeper test surgery.
- No new tests verifying the ALS-bridged path actually rebinds the context — the comment at `instance.ts:19-20` claims this, but there's no regression test for "Instance.directory read inside init sees the right value". Add one if the bridge is the actual point of the PR.

## Verdict

`merge-after-nits` — type migration is safe and the boilerplate is bounded to test files. Address the helper-extraction nit and add one ALS-binding regression test before merge.
