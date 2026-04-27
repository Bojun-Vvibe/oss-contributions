# sst/opencode PR #24626 — fix(httpapi): mount workspace bridge routes

- **PR**: https://github.com/sst/opencode/pull/24626
- **HEAD**: `88a4714b`
- **Touch**: 4 files, +14/−10

## Context

`#24589` (the multi-root workspaces PR reviewed in drip-113) shipped a new
`WorkspacePaths` route module under `server/routes/instance/httpapi/workspace.ts`
plus an integration test, but never *registered* those handlers in
`InstanceRoutes`. The routes were implemented and tested in isolation through
`ExperimentalHttpApiServer.webHandler()` but no caller of `InstanceRoutes`
could reach them: any GET/POST to `/workspaces`, `/workspaces/:id/status`,
etc. would return 404 in production. This PR is the trailing fixup.

## Diff

`server/routes/instance/index.ts:28` adds the import; `:148-153` mounts the
six handlers (`adaptors`, `list` GET+POST, `status`, `remove` DELETE,
`sessionRestore` POST). The route surface matches the WorkspacePaths
contract exported by the workspace module — every path declared there is
now registered.

The test rewrite at `test/server/httpapi-workspace.test.ts:1-65` is the
more interesting half. The previous test went through
`ExperimentalHttpApiServer.webHandler()` directly with an empty
`Context.empty()` — which is exactly why the missing route registration
slipped through CI: the test bypassed the registration layer entirely.
The new shape:

1. flips `Flag.OPENCODE_EXPERIMENTAL_HTTPAPI = true` at request time
   (with `originalHttpApi` saved/restored in `afterEach:60`),
2. injects a stub `UpgradeWebSocket` (returns `501` — fine, none of the
   workspace routes use ws), and
3. routes through `InstanceRoutes(websocket).request(path, init)`.

That means the test would now **catch** the original "route declared but
not mounted" bug, which is the right shape for a fixup-test pair.

## Risks / nits

- The two zod-parse strips in `config.ts:69-72` and `project.ts:102` are
  bundled with the routing fix but are unrelated. They're trusting the
  Effect HttpApi schema layer to have validated the payload one level up —
  probably correct given the `Layer.unwrap(Effect.gen(...))` shape, but
  worth a one-line commit note explaining "double-validation drop, schema
  layer already enforced it".
- No regression test pinning that `WorkspacePaths.list` is wired via both
  `app.get` and `app.post`; a follow-up PR removing one would silently
  pass the existing tests if those tests only exercise the half that's
  still routed.
- `WorkspacePaths.sessionRestore` is mounted as POST only — confirm that
  matches the intended REST semantics (it could also reasonably be PUT).

## Verdict

Verdict: merge-as-is
