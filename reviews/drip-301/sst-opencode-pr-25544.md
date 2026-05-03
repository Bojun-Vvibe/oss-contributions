# sst/opencode PR #25544 — sketch: typed SessionNotFound (design validation)

- Author: kitlangton
- Head SHA: `4837bb8904a483e5151887fcaa323f7525a228d9`
- State: DRAFT — explicitly "design sketch, NOT to be merged as-is"
- Diff: +135 / -59 across server route/handler files

## Observations

1. **`groups/session.ts:123-130` declares `error: [HttpApiError.BadRequest, Session.SessionNotFound]`** instead of the generic `HttpApiError.NotFound`. This is the central proposal — the typed error class carries an `httpApiStatus: 404` annotation and a `data: { message }` schema, so HttpApi auto-routes the status code and serializes the body without a wrapper. The PR text confirms wire-shape parity with the legacy Hono `NamedError` envelope (`{ name, data: { message } }`).
2. **`handlers/session.ts:41-56` extends `mapNotFound`** to ALSO catch `Session.SessionNotFound` and rewrap it as `HttpApiError.NotFound`. This is the deliberate transitional shim: handlers whose endpoint declarations have *not* been migrated still call `session.get()` internally, and without this catch their typed error would surface as a 500. Smart — it lets the migration proceed one endpoint at a time without a flag day.
3. **`handlers/session.ts:90-94` removes the `mapNotFound` wrapper from `get`** while leaving it in place for `update`, `share`, `unshare` (which now wrap a larger `Effect.gen` block — see `handlers/session.ts:190-237`). The asymmetry is correct given the sketch scope, but the diff makes those callsites noisier than they need to be; a follow-up should likely either migrate the inner `session.get()` calls to use `Effect.catchTag("SessionNotFound", ...)` or migrate those endpoints' error declarations too.
4. **The cast `as Effect.Effect<A, Exclude<E, Session.SessionNotFound> | HttpApiError.NotFound, R>`** at the end of `mapNotFound` is a type-inference workaround. Worth a `// TODO: drop cast once all session.get callers declare SessionNotFound` so future readers know it's load-bearing only during the migration.
5. **Hono adapter (`trace.ts`, per PR description)** adds `adaptTypedErrors` that catches `SessionNotFound` and re-throws the legacy `NotFoundError` defect — keeps the existing `ErrorMiddleware` rendering 404 on the Hono path. Good belt-and-suspenders for the parity period; should be deleted in lockstep with `openapiHono()` removal (see drip-300 PR #25545).

## Verdict: `needs-discussion`

This is a draft labelled by the author as "design validation, not for merge", and the design choice is sound — typed errors at the service boundary is the right direction and the schema annotations carry the wire format faithfully. Discussion items before this becomes a real PR: (a) commit to whether `SessionNotFound` lives in `Session` or a shared `errors/` module so other services can follow the same pattern; (b) decide whether the `mapNotFound` shim is permanent (handlers that internally call other services' `get()` will keep needing it) or transitional; (c) confirm SDK regen on the typed-error path produces the same TypeScript shape clients consume today.
