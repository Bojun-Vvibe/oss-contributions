# sst/opencode #24168 — Refactor HttpApi auth middleware wiring

**Link:** https://github.com/sst/opencode/pull/24168
**Tag:** refactor, auth-surface

## What it does

Carves the experimental HttpApi authorization middleware out of the
central server composer and into a shared
`server/routes/instance/httpapi/auth.ts` module, then attaches it
per-route via `.middleware(Authorization)` on each `HttpApiGroup`
(`config`, `file`, `mcp`, `permission`, …). Two security schemes are
declared on the `HttpApiMiddleware.Service`: `basic` (HTTP Basic) and
`authToken` — the latter modeled as an `HttpApiSecurity.apiKey({ in:
"query", key: "auth_token" })` instead of the previous request-header
rewrite. Validation logic is unchanged: compare against
`Flag.OPENCODE_SERVER_USERNAME`/`PASSWORD`, fall through if no password
is configured, and fail with a tagged `Unauthorized` (httpApiStatus 401).

## What it gets right

- **Per-route attachment** is the right primitive: it makes
  authorization visible at the route definition site instead of being a
  composer-wide invisible wrap, and it's easy to grep for which routes
  forgot the middleware.
- Tagged error class with `httpApiStatus: 401` keeps the OpenAPI surface
  honest (the 401 response shape comes for free).
- Modeling `auth_token` as an OpenAPI security scheme rather than a
  header rewrite means the generated spec/SDK actually advertises the
  parameter — clients no longer have to know the magic out-of-band.
- The `decodeCredential` helper degrades gracefully on malformed base64
  (returns `emptyCredential`, which then hits the username/password
  mismatch branch) — no panic on garbage input.

## Concerns / risks

- **Per-route opt-in is a footgun.** Future route authors will forget
  `.middleware(Authorization)` and ship an unauthenticated endpoint. A
  composer-level default-deny + per-route opt-out would be safer; the
  current shape is "default-public, hope you remember". At minimum a
  test that asserts every registered HttpApiGroup carries Authorization
  would close the gap.
- The empty-password short-circuit (`if (!Flag.OPENCODE_SERVER_PASSWORD)
  return yield* effect`) means the experimental HttpApi is wide open by
  default. That's consistent with prior behavior but worth flagging in
  the route docs annotation.
- `auth_token` in a query parameter will leak into access logs and
  shell history. Header-based bearer would be safer; if query is
  load-bearing for browser fetch reasons, document the leak hazard.
- `username !== (Flag.OPENCODE_SERVER_USERNAME ?? "opencode")` performs
  a non-constant-time compare; for a localhost dev surface this is fine
  but worth noting if this ever ships beyond loopback.

## Suggestion

Add a `httpapi.test.ts` that constructs the composed `HttpApi` and
asserts every group has `Authorization` middleware bound. Cheap insurance
against the per-route opt-in footgun.

**Tag:** auth-surface, default-deny-gap
