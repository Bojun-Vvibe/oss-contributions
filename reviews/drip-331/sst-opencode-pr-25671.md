# sst/opencode #25671 — fix(server): read auth Config from Flag for HttpApi/Hono parity

- SHA: `da5e29b3206ce2d557290cbac917a350c364fe3e`
- State: OPEN (Draft per body), +65/-50 across 5 files

## Summary

`ServerAuth.Config` previously inherited Effect `ConfigService.defaultLayer`, which reads `EffectConfig.string("OPENCODE_SERVER_PASSWORD")` once and memoizes via Layer identity. Runtime mutation of `process.env` or `Flag.OPENCODE_SERVER_PASSWORD` was therefore invisible to the HttpApi auth middleware, while Hono's middleware reads `Flag.*` per request. PR overrides `defaultLayer` with `Layer.sync` so each fresh listener (each `memoMap`) snapshots the current `Flag.*` values.

## Notes

- `packages/opencode/src/server/auth.ts:30-39` — `static override get defaultLayer()` returning `Layer.sync` is the right shape for "re-read at layer-build time," and the `Option.some / Option.none` mapping for `password` matches the original `EffectConfig.option` semantics. Field schemas are still declared on the class so explicit `ServerAuth.Config.layer({...})` overrides in tests keep working — good migration story.
- `packages/opencode/src/server/auth.ts:17-29` — the inline comment block is unusually long (~12 lines) but earns its keep: it documents the per-listener vs per-request gap explicitly. Consider promoting "consider reading `Flag.*` directly in middleware for per-request parity" to a tracked TODO/issue link, otherwise it'll get lost on the next refactor.
- `packages/opencode/test/server/httpapi-listen.test.ts:260-275` — parameterizing the no-auth PTY test over both backends is the right regression lock. Without this, the bug recurs silently because the previous fixture forced `OPENCODE_EXPERIMENTAL_HTTPAPI = false`. Suggest adding a second positive case: start a listener *with* a password, mutate `Flag.OPENCODE_SERVER_PASSWORD = undefined` in-process, restart, and confirm 200 OK — that exercises the cache-invalidation invariant directly.
- `packages/opencode/test/server/httpapi-raw-route-auth.test.ts:14-19, 44-47` — switching from `ConfigProvider.fromUnknown(...)` to direct `Flag` mutation with full `original` snapshot/restore in `afterEach` is correct and matches production read semantics. Good catch on tracking all three Flag fields, not just the one you set.
- `packages/opencode/test/server/httpapi-ui.test.ts:50-58, 72-90` — replacing the `ConfigProvider` plumbing with `ServerAuth.Config.layer({...})` cleanly. Slight smell: the `uiApp` helper constructs `authConfigLayer` outside the `HttpRouter.use` block; if `serveUIEffect` ever needs the same config, you'd want it threaded through both branches. Not blocking — current tests don't exercise that path.

## Verdict

`merge-after-nits` — diagnosis is precise (`Layer` identity memoization vs runtime mutation) and the fix is minimally invasive. The PR is correctly marked Draft pending an architectural call on whether `ConfigService` itself should stop offering Effect-Config defaults for runtime-mutable fields. Address the per-listener vs per-request TODO with a tracking issue link, add the positive cache-invalidation regression test, then merge.
