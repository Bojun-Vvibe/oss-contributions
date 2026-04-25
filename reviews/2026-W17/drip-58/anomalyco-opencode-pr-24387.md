# anomalyco/opencode#24387 — feat(httpapi): bridge config update endpoint

- **Repo:** anomalyco/opencode
- **Author:** kitlangton (Kit Langton)
- **Head SHA:** `90e163ee` (90e163eef84e6c305f145e3b81da15c0f9654a5a)
- **Size:** +111 / -5

## Summary
Migrates the `PATCH /config` mutation from Hono to the Effect `HttpApi` bridge.
Adds a `dispose` option to `Config.update` so the route handler can defer
instance teardown to a post-response lifecycle hook
(`markInstanceForDisposal`) instead of inline.

## Specific references
- `packages/opencode/src/config/config.ts:283` @ `90e163ee` — `update` signature gains `options?: { dispose?: boolean }`. Default behavior (`dispose !== false`) preserves the legacy contract for non-bridge callers. Good.
- `packages/opencode/src/config/config.ts:722-729` @ `90e163ee` — `if (options?.dispose !== false) yield* Effect.promise(() => Instance.dispose())`. Correctly falls through to dispose when no options are supplied.
- `packages/opencode/src/server/routes/instance/httpapi/config.ts:24-33` @ `90e163ee` — adds `HttpApiEndpoint.patch("update", root, { payload: Config.Info, success: Config.Info })`. Schema parity with the Hono original needs verification (see #1 below).
- `packages/opencode/src/server/routes/instance/httpapi/config.ts:69-74` @ `90e163ee` — `update` handler does `Config.Info.zod.parse(ctx.payload)`, calls `configSvc.update(payload, { dispose: false })`, then `markInstanceForDisposal`. Good ordering: write succeeds before marking disposal.
- `packages/opencode/specs/effect/http-api.md:46-58` @ `90e163ee` — new "Hono Deletion Checklist" section. Item 9 explicitly calls out the disposal-via-lifecycle pattern this PR introduces — solid doc-as-spec hygiene.

## Observations
1. **Double-validation**: `HttpApiEndpoint.patch` already declares `payload: Config.Info` so Effect schema-validates on the way in. Then the handler does `Config.Info.zod.parse(ctx.payload)` again (`config.ts:71`). Either (a) that's because the bridge passes the raw payload pre-validation (in which case fine), or (b) it's redundant. Worth a comment explaining which.
2. **Return value is the payload, not the merged result**: handler returns `payload` directly (`config.ts:74`). But `Config.update` does `mergeDeep(existing, config)` and writes the merged form — so the persisted file != the response body. Consumers who PATCH a partial config will get back their partial input rather than the new effective config. Either re-read after write, or document this.
3. **Checklist self-compliance**: spec checklist item 3 says "tests cover both the default `HttpApi` path and the fallback Hono path until the fallback is removed." I don't see new bridge tests in this diff. The `config | bridged` table row suggests it's mounted by default — which makes the missing test more important.

## Verdict
**merge-after-nits** — clarify return-value semantics (#2) and add at least one bridge test for the new mutation path before relying on it as the default.
