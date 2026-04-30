# Review: sst/opencode#25154 — Fix HttpApi raw route authorization

- PR: https://github.com/sst/opencode/pull/25154
- Head SHA: `4c791b00e5f4b76d9569b33eb886bf7db174b0b3`
- Author: kitlangton
- Files: 5 changed (+176/-14)
- Merged: 2026-04-30

## Stated purpose
Close an authorization gap where the *raw* HttpApi instance routes (`/event` SSE stream and `/pty/:ptyID/connect` WebSocket upgrade) bypassed the existing query-param `auth_token` policy that protected the codegen'd HttpApi handlers. Also restore log parity on sync replay so operators don't lose the per-request observability that the legacy Hono route used to emit.

## What actually changed
1. **`middleware/authorization.ts`** — the validation logic is split into two pure predicates `isAuthRequired(config)` and `isCredentialAuthorized(credential, config)` (lines 47-60), then a *new* synchronous `validateRawCredential` (lines 78-87) returns a 401 `HttpServerResponse.empty` directly instead of going through `HttpApiError.Unauthorized` (which only the codegen layer knows how to map). The new `authorizationRouterMiddleware` (lines 89+) is a `HttpRouter.middleware()`-shaped wrapper that decodes the `auth_token` query param via `HttpServerRequest`, builds the bearer credential, and runs `validateRawCredential` — gating the raw routes the same way the codegen middleware gates the typed ones.
2. **`server.ts`** — wires the new router middleware into the raw route mount (4 added / 3 removed) so `/event` and `/pty/:ptyID/connect` see the gate before reaching the handler.
3. **`handlers/sync.ts`** — adds two `log.info("sync replay requested" / "sync replay complete", { sessionID, events, first, last, ... })` bookends around `SyncEvent.replayAll(events)` (lines 41-54) — pure observability, no behaviour change. The `directory: ctx.payload.directory` field on the request side is the load-bearing breadcrumb that lets ops correlate replays back to a specific workspace.
4. **`test/server/httpapi-raw-route-auth.test.ts`** — new 89-line test file that boots the server with `auth_token=...` configured and asserts both raw routes return 401 without the param and 200/upgrade with it.

## Quality / risk observation
The split into `isAuthRequired` + `isCredentialAuthorized` is the right shape — the bug was *exactly* the kind that arises when an auth check is fused into a single procedural function: the raw-route path needed the same policy but couldn't reuse it because the original was an `Effect`-shaped helper that returned `HttpApiError.Unauthorized`. Extracting the predicates first, then writing a sync-shaped `validateRawCredential` that satisfies the `HttpRouter.middleware` contract, is cleaner than the alternative of either (a) duplicating the password-check logic or (b) trying to bridge `Effect` semantics into the raw-route path. One minor nit: `validateRawCredential` passes the `effect` parameter through unchanged when auth isn't required *or* when the credential is good, which is correct, but the `Effect.Effect<A, E, R>` generic on a function that no longer wraps anything in `Effect.gen` is a small API smell — it would read more naturally as `(effect: Eff, credential, config) => Eff` since no Effect-flavoured composition happens inside. Zero risk to ship as-is. Coverage at 89 new test lines for two new failure modes is appropriately scoped — this is exactly the regression net the previous version was missing.

## Verdict
`merge-as-is`
