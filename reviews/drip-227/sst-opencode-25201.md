---
pr: sst/opencode#25201
title: "Pass CORS options to HttpApi backend"
head_sha: 93b7bdb004af9aca091cc0e3d8eaa7ec211b463e
verdict: merge-after-nits
drip: drip-227
---

## Context

`Server.listen({ cors: [...] })` historically only fed custom origins
into the Hono backend. Once the Effect HttpApi backend was added,
`createHttpApi()` ignored caller-supplied origins, so a user starting
the new backend with a custom CORS allowlist silently got the default.
This PR plumbs the option through.

## Diff walkthrough

`server/cors.ts:3-7` introduces a shared `CorsOptions = { readonly cors?:
ReadonlyArray<string> }` type and re-exports it; both Hono middleware
(`server/middleware.ts:14,70`) and the HttpApi route layer
(`routes/instance/httpapi/server.ts:41,80-86`) now consume the same shape
instead of two ad-hoc inline shapes.

The load-bearing change is at `routes/instance/httpapi/server.ts:80-86`:
the `cors` middleware is no longer a static `Layer` but a function
`(corsOptions?: CorsOptions) => HttpRouter.middleware(...)` so the
allowed-origin predicate closes over the caller's list. `createRoutes`
(`:134-181`) wraps the previous static `routes` `Layer.mergeAll(...)`
call so the CORS layer can be parameterized per call. The original
`export const routes = createRoutes()` at `:181` preserves the no-args
default for existing callers.

`webHandler` (`:185-193`) is split into `defaultWebHandler` (memoized,
no-arg) and a public `webHandler(corsOptions?)` that branches: when no
custom origins are passed, return the cached default; otherwise build a
fresh `Layer.makeMemoMapUnsafe()` so the per-instance CORS layer doesn't
leak into the global memo.

In `server.ts:42-47` `ListenOptions = CorsOptions & {port,hostname,...}`
collapses three call-site shapes; `createHttpApi(opts)` (`:85-87`) now
forwards them. Test at `httpapi-cors.test.ts:65-89` boots
`Server.listen({ cors: ["https://custom.example"] })` and asserts the
preflight `access-control-allow-origin` matches — exact-string contract
pin against future regressions.

## Risks / nits

- `Layer.makeMemoMapUnsafe()` per call (`server.ts:191`) means every
  custom-CORS startup builds a fresh layer graph; fine for the listen
  path (one-time at boot), but worth a comment noting why it's safe to
  skip the memo here.
- The new `CorsOptions` type uses `ReadonlyArray` but the consumer
  `isAllowedCorsOrigin` (`cors.ts`) iterates without mutation, so the
  variance is consistent — good.
- No test exercises the `memoMap` reuse path (default-args case); the
  parity test only covers the custom branch. A second arm asserting that
  two `webHandler()` calls return the same instance would lock in the
  cache invariant.

## Verdict

**merge-after-nits** — straightforward, type-safe plumbing fix with a
proper end-to-end regression test. Address the memo comment + add the
default-cache parity arm and ship.
