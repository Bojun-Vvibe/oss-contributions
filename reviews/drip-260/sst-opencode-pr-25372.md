# sst/opencode PR #25372 — Extract InstanceStore.provide helper

- PR: https://github.com/sst/opencode/pull/25372
- Head SHA: `b96436600302a57f2e66364f763326554b510b0c`
- Author: @kitlangton
- Size: +10 / -13
- Status: MERGED

## Summary

Pulls the `store.load(input)` + `Effect.provideService(InstanceRef, ctx)` pattern out of the HttpApi instance-context middleware into `InstanceStore.provide(input, effect)`. Net -3 LOC and the middleware no longer constructs `InstanceContext` directly. Motivation is to give CLI/background callers a one-liner instead of having every Effect-native call site copy the load+provide pair.

## Specific references

- `packages/opencode/src/project/instance-store.ts:23` — adds `provide` to the `Interface`, with the right variance: `<A, E, R>(input, Effect<A, E, R>) => Effect<A, E, R>`. Doesn't widen the requirement set, which is what you want.
- `packages/opencode/src/project/instance-store.ts:173-174` — implementation: `load(input).pipe(Effect.flatMap((ctx) => effect.pipe(Effect.provideService(InstanceRef, ctx))))`. Direct mechanical extraction.
- `packages/opencode/src/server/routes/instance/httpapi/middleware/instance-context.ts:25-35` — middleware collapses from `makeInstanceContext` + manual `provideService(InstanceRef, ctx)` to one `store.provide(...)` call wrapping the `provideService(WorkspaceRef, ...)`. The dead `makeInstanceContext` helper is removed.

## Verdict: `merge-as-is`

Pure refactor with no behavior change. Test claim (`15/15 pass` on the two relevant suites) is plausible — the only externally observable surface is `InstanceStore.Interface`, which gained a method, and the middleware, whose effect type is unchanged. Net negative LOC.

## Nits (non-blocking)

1. `provide` does not appear in the `Service.of({ load, reload, dispose, disposeAll, provide })` ordering grouping — it's appended last. Fine, but if there's an alphabetical or "lifecycle vs. usage" convention elsewhere in the codebase, worth aligning.
2. Future callers should be aware: `provide` runs `load` (which can fail) inside the wrapped effect's error channel, but the new signature's `E` is only the wrapped effect's error type. `load`'s failures (e.g. `init` rejecting) will surface as a defect rather than a typed error. Worth a doc-comment, especially before CLI commands start adopting it.
