# sst/opencode #25027 — test: cover HttpApi workspace routing middleware

- **URL:** https://github.com/sst/opencode/pull/25027
- **Head SHA:** `44f5a939bcabf0bfbdafbbe7cd80ea476fbd1004`
- **Files:** `packages/opencode/src/effect/service-use.ts` (new, +38), `packages/opencode/src/project/project.ts` (+3/-1), `packages/opencode/test/server/httpapi-workspace-routing.test.ts` (new, +437), `packages/opencode/test/server/workspace-proxy.test.ts` (+1/-1)
- **Verdict:** `merge-as-is`

## What changed

Three things bundled into one PR:

1. **`serviceUse` helper** at `packages/opencode/src/effect/service-use.ts:23-44` — a `Proxy`-backed accessor that turns any Effect-Context Service into a `tag.use(svc => svc.method(...args))` shape with the dependency `R | Identifier` propagated through TS mapped types at `:11-21`. Two `oxlint-disable-next-line typescript-eslint/no-unsafe-type-assertion` comments at `:33` and `:36` carry the load-bearing rationale ("Proxy keys are checked at runtime", "ServiceUse exposes only Effect-returning methods").
2. **`Project.use` accessor** at `project.ts:489` (`export const use = serviceUse(Service)`) — exposes the Project service through the new helper. The drive-by at `:182` (`Effect.map((x) => ProjectID.make(x))`) wraps the previously bare reference to make the closure shape explicit; functionally identical but stops a subtle "method-as-callback" inference issue when `ProjectID.make` is called via `Effect.map`.
3. **New `httpapi-workspace-routing.test.ts`** — 7 `it.live` cases covering the routing matrix that was previously untested:
   - `:294-358` — remote workspace HTTP proxying through `HttpApiProxy.http`, asserting the second-server `forwarded?.headers["x-target-auth"]` propagates and `x-opencode-directory` / `x-opencode-workspace` get stripped before forward (`:356-357`).
   - `:361-383` — 503 on insert-without-sync via `insertRemoteWorkspaceWithoutSync` at `:233-244` (DB row exists but `Workspace.isSyncing` is false), with the exact body text `broken sync connection for workspace: ${workspaceID}`.
   - `:385-420` — WebSocket proxy via `Socket.makeWebSocket` to a `listenRemoteWebSocket()` upstream that emits `protocol:chat` and `echo:hello` through `Queue.unbounded<string>`.
   - `:422-438` — 500 on unknown workspace ID using `WorkspaceID.ascending("wrk_missing")` so the middleware error response is what reaches the client (handler never runs).
   - `:440-468` — control-plane carve-out: `GET /session?workspace=...` stays local (returns `process.cwd()` directory) even when a workspace is selected.
   - `:470-490` — directory query/header fallback path (`?directory=...` and `x-opencode-directory: ...`) when no workspace selected.
   - `:492-516` — local workspace swaps `WorkspaceRouteContext.directory` to the workspace target dir.

## Why it's right

- The previous coverage in `workspace-proxy.test.ts` only exercised the `HttpApiProxy.websocket` happy path (drip-189 reviewed PR #25017). This PR fills the *middleware* coverage gap above the proxy: routing decisions, error response shapes, and control-plane carve-out — exactly the layer that decides whether a request becomes a proxy call at all.
- The `serviceUse` helper isolates the `Proxy`-based dynamic boundary into one 38-line module with two narrow `oxlint-disable` annotations and TS mapped types that *only* expose `Effect`-returning methods (`Shape[Key] extends EffectMethod ? Key : never` at `:13`). Future Service consumers get a typed callsite without each touching the unsafe `as keyof Shape` cast.
- The `localAdaptor`/`remoteAdaptor` factories at `:162-182` plus the `createWorkspace` `Effect.acquireRelease` at `:196-208` keep test workspaces scoped — no cross-test bleed via the `registerAdaptor` global. The `testStateLayer` at `:112-123` flips `Flag.OPENCODE_EXPERIMENTAL_WORKSPACES` and resets the DB inside an `Effect.addFinalizer`, so even if a test panics the global flag goes back.
- The `syncResponse` helper at `:189-194` deserves explicit call-out: by stubbing `/base/global/event` (SSE) and `/base/sync/history` on the upstream, `Workspace.isSyncing` resolves true in the test without requiring real sync infrastructure. The 503 case at `:361-383` deliberately *skips* this stub via `insertRemoteWorkspaceWithoutSync` — that's the difference between the two paths.
- Drive-by at `workspace-proxy.test.ts:49` (`echo:${String(message)}`) silences a TS-strict `[object Object]`-coerce warning when `message` is `Uint8Array` and matches the same pattern in the new `echoWebSocket` at `:273`.

## Nits / not blockers

- The `serviceUse` `Proxy` `get` handler at `:29-39` only validates `typeof key === "string"`; a future Service consumer that accidentally introspects `Symbol.iterator` (e.g. via `for..of` on the proxy) would silently get `undefined`. A one-line `if (typeof key === "symbol") return undefined` carve-out would make the failure mode explicit, but no current call path hits this.
- The 500 status assertion at `:435` for "unknown workspace ID" is correct relative to the current middleware behavior, but a 404 would be the conventional shape — worth a follow-up issue to either rationalize the status code or document why 500 is intentional (probably "fail loud, not absent — a missing workspace shouldn't be confused with `/probe` not existing").
- The `it.live` modifier means these tests bind real OS sockets via `NodeHttpServer.layerTest` + `listenAdditionalServer`. Running them in parallel might have port-pressure issues on CI; the `port: 0` at `:156` mitigates this but a per-test scope finalizer to drain the additional server isn't visible in the diff slice.

## Risk

Test-only change plus a tiny functionally-identical refactor at `project.ts:182`. The new `serviceUse` accessor has zero current consumers beyond the `export const use = serviceUse(Service)` at `project.ts:489`, which is itself unused in the diff slice — safe to land. The `String(message)` cast on the existing test is a no-op for string payloads. Worst case if the new tests are flaky on CI, they get marked `.skip` without affecting production code.
