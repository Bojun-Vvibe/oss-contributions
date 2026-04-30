# sst/opencode #25033 — test: cover HttpApi authorization middleware

- **URL:** https://github.com/sst/opencode/pull/25033
- **Head SHA:** `5993cef05ea7cadc4250bf97ef0c6c1e1eb5e102`
- **Merge SHA:** `38adc13295471de6a8c84bc73d2c94b8e294905e`
- **Files:** `packages/opencode/test/server/httpapi-authorization.test.ts` (new, +137)
- **Verdict:** `merge-after-nits`

## What changed

Adds five `it.live` cases against a minimal in-process HttpApi (`Api`/`handlers`/`apiLayer` at `:9-23`) wired with `authorizationLayer` from the production middleware. The test app exposes a single `GET /probe → "ok"` endpoint behind the `Authorization` middleware so each case exercises the real 401/200 path through the platform `HttpApi` machinery.

Coverage:

1. **`allows requests when server password is not configured`** at `:74-81` — exercises the bypass branch when `Flag.OPENCODE_SERVER_PASSWORD` is `undefined`, asserts `200` + body `"ok"`.
2. **`requires configured password for basic auth`** at `:83-99` — sets the password via `useAuth({ password: "secret" })`, fires three concurrent requests (`missing`, `badPassword`, `good`), asserts `401/401/200` respectively. The `concurrency: "unbounded"` in `Effect.all` is correct here because the middleware reads the layer-bound config, not request state.
3. **`respects configured basic auth username`** at `:101-112` — proves the default `"opencode"` username is rejected when a non-default username is configured (`"kit"`), and the configured one succeeds.
4. **`accepts auth token query credentials`** at `:114-122` — exercises the `?auth_token=<base64(user:pass)>` query-credential alternative path that some clients use when `Authorization` headers are awkward (e.g. WebSocket upgrades, `EventSource`).
5. **`rejects malformed auth token query credentials`** at `:124-130` — asserts `not-base64` produces `401` rather than tripping a decode panic.

Test scaffolding uses the same `testStateLayer` + `useAuth` `Effect.acquireRelease` pattern that #25035 then *replaces* — this PR was the "lock the behavior with tests, then refactor under green" prerequisite for the `ConfigService`-based rewrite.

## Why it's right

- **Tests the public HTTP contract through real `HttpApi.make` + `HttpRouter.serve`.** The middleware is exercised end-to-end via `HttpClient.execute`, not via a unit test against `validateCredential` in isolation. That means the test catches integration regressions in routing, header parsing, and `HttpApiError.UnauthorizedNoContent` mapping — all of which would silently break a unit-only test.
- **`it.live` (not `it.effect`) is the right choice for HTTP tests.** `it.live` runs against the live Effect runtime including the `NodeHttpServer.layerTest`-bound socket, while `it.effect` would short-circuit time/IO. The five cases correctly use `live` because they send real bytes through a real loopback socket.
- **Five cases cover the complete decision tree of the middleware** — bypass-when-unconfigured, configured-password-with-three-credential-states, configured-username-with-default-rejection, query-credential-happy-path, query-credential-malformed-rejection. The only branch not exercised is "configured password with empty string" (which the middleware treats as bypass per `authorization.ts:38` post-#25035), and that's covered indirectly by case 1.
- **Locks behavior before the #25035 refactor.** This PR is the `Add tests so we can refactor safely` half of a two-step. Merging it first means `#25035`'s diff lands under a green suite that already proves the public contract — exactly the right ordering.

## Tiny nits (non-blocking)

- The `useAuth` helper at `:48-67` mutates `Flag.OPENCODE_SERVER_PASSWORD` / `Flag.OPENCODE_SERVER_USERNAME` directly with `Effect.acquireRelease`. That's necessary today because `authorizationLayer` reads the global, but it means **two `it.live` cases running concurrently against the same module would race on the global**. This file's cases are sequential within `describe`, so it's fine in practice — but it's worth a one-line comment naming the constraint so a future "let's parallelize this suite" doesn't silently break. (Side note: #25035's layer-injected version of the middleware fixes this hazard at the root by removing the global entirely; once that PR is merged, this `useAuth` ceremony goes away.)
- The `testStateLayer` at `:25-41` and `useAuth` at `:48-67` duplicate the "save → mutate → restore in finalizer" pattern. The `testStateLayer` is the "always start with both Flags `undefined`" base case, and `useAuth` is the "set them to specific values inside one test" override — but the duplication means both have to be maintained in lockstep if the field set ever grows. A single helper `withFlags(input)` returning an `Effect<void, never, Scope>` would express both shapes uniformly.
- The `basic` helper at `:43-44` and the `token` helper at `:46` both `Buffer.from(...).toString("base64")` — could be one helper that the `basic` form wraps with the `Basic ${...}` prefix. Pure ergonomics; no behavioral risk.
