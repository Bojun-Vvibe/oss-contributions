# sst/opencode#24626 — fix(httpapi): mount workspace bridge routes

- PR: https://github.com/sst/opencode/pull/24626
- Head SHA: `88a4714b`
- Diff: +18 / -14 across 4 files

## What it does
Wires the existing workspace HttpApi paths through the experimental Hono bridge in `packages/opencode/src/server/routes/instance/index.ts:148-153` (six new `app.{get,post,delete}` registrations for `WorkspacePaths.{adaptors,list,status,remove,sessionRestore}`). Also drops redundant `Zod.parse` re-validation of already-decoded payloads in `httpapi/config.ts:70` (Config update) and `httpapi/project.ts:102` (Project update). The workspace test (`test/server/httpapi-workspace.test.ts`) is rewritten to drive `InstanceRoutes` instead of `ExperimentalHttpApiServer.webHandler()` directly, which makes it actually exercise the same path users hit.

## Strengths
- Test refactor (`request()` helper at `httpapi-workspace.test.ts:23-32`) flips the experimental flag inside the helper and uses a stub `UpgradeWebSocket`, so the test now validates the real wire-up rather than the bypass.
- Removing the second `Zod.parse` is correct: the HttpApi layer already decodes against the schema before the handler runs; re-parsing only burns cycles and risks divergence if the schema gains transforms.

## Concerns
- The new route registrations are flat `app.get/post/...` calls; if `WorkspacePaths` ever adds a path it must be remembered to wire it here too — there is no compile-time guarantee. A small `for (const [method, path] of WorkspacePaths.entries)` loop, or a typed registry shared with the test, would make this future-proof.
- `httpapi-workspace.test.ts` flips `Flag.OPENCODE_EXPERIMENTAL_HTTPAPI = true` in `request()` on every call but only restores it in `afterEach`. That's fine within this file, but if any other test in the same Bun process runs before the `afterEach` fires, it will see the flag flipped. Setting it once in a `beforeAll` would be cleaner.
- The PR doesn't show a regression test asserting that the routes are reachable (e.g. an explicit 200 vs 404 for `WorkspacePaths.list`). The changed test exercises them implicitly via the existing assertions, which is acceptable but not as defensive.

## Verdict
**merge-after-nits** — landing the route wiring + Zod cleanup is a clear win; consider the registry/loop suggestion in a follow-up rather than blocking on it.
