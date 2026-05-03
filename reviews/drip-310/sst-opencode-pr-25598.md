# sst/opencode #25598 — feat(server): run effect-httpapi backend on Effect's BunHttpServer

- PR: https://github.com/sst/opencode/pull/25598
- Author: kitlangton
- Head SHA: `280aab971bff9f8f59cdc0c3dbb9994ba52e4516`
- Updated: 2026-05-03T14:03:52Z

## Summary
Replaces the hand-rolled `packages/opencode/src/server/httpapi-listener.ts` (244 LOC) with `BunHttpServer.layer` from `@effect/platform-bun`. When the `effect-httpapi` backend is selected, `Server.listen()` now drives HttpApi routes through `BunHttpServer` + `HttpRouter.serve` directly, deleting the inline PTY/workspace-proxy WS plumbing because the existing HttpApi-side `HttpApiProxy.websocket` and `ptyConnectRoute` already implement those flows correctly once `request.upgrade` actually works. Net −258 LOC. Closes #25594, #25595; deletes the PoC from #25547.

## Observations
- `packages/opencode/package.json` adds `@effect/platform-bun` at the same `4.0.0-beta.57` pin as the rest of the `@effect/*` catalog — good, no version skew, and the catalog entry in the root `package.json` is updated in the same PR. Diff is minimal and mechanical.
- `packages/opencode/src/server/server.ts` `listen()`: the dispatch via `selected.backend === "effect-httpapi" ? listenHttpApi(...) : listenLegacy(...)` cleanly separates the two paths, and the outer `listen()` wrapper (mDNS publish, URL construction, idempotent stop) is shared. Stop handlers are idempotent via the `closing` promise — good.
- `listenHttpApi` builds a `Layer` per call: `HttpRouter.serve(...).pipe(Layer.provideMerge(BunHttpServer.layer({ port, hostname })))`. Two things to double-check:
  1. The PR description says a *fresh memo map is used when `opts.cors` is non-empty (mirrors existing `webHandler`)*. Verify this is wired in the new path and not lost in the cutover — losing the per-CORS memo would cause every CORS-configured listen() call to recompile the runtime. Inspect the full `listenHttpApi` body (truncated in the diff snapshot here at line ~70) for the `memoMap` usage.
  2. The `HttpMiddleware` `R = any` cast at the layer boundary called out in the description: an `as any` (or equivalent unchecked cast) survives review easily but should be commented inline pointing at the `HttpMiddleware` interface design issue, so the next person doesn't try to "clean it up" and break the build.
- Port-allocation behavior is documented as "try 4096 first, fall back to 0" when `opts.port === 0`. Confirm the new code reads back the bun-allocated port via `HttpServer.HttpServer` from the scope; from the diff this is implied but truncated. A unit test or even a `console.log` smoke run pinning the port semantics would be cheap insurance — this is the kind of thing that breaks silently on Linux/Bun version bumps.
- Deleting `httpapi-listener.test.ts` (109 LOC) is a meaningful test-surface loss. Even though the listener it tested no longer exists, the PR should add (or the PR description should call out an existing) end-to-end test that exercises `request.upgrade` through the new path — at minimum a test that opens a PTY connect WS against `BunHttpServer.layer` and reads back a byte. Without that, the next refactor in `BunHttpServer` upstream (`platform-bun` is `4.0.0-beta.*`) can silently regress PTY without anyone noticing in CI.
- `Server.listen()` legacy backend is preserved untouched — good, this is an additive cutover and rolling back is just flipping the backend selector.
- Effect platform-bun is `4.0.0-beta.57` — beta. The opencode catalog already commits to that beta line, so this PR is not introducing new instability, just expanding the surface that depends on it.

## Verdict
`merge-after-nits`
