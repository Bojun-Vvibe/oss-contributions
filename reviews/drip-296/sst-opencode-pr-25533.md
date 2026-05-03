# Review: sst/opencode PR #25533

- **Title:** fix(httpapi): 404 status + body shape parity for missing sessions
- **Author:** kitlangton (Kit Langton)
- **Head SHA:** `98fef4555364ea76163ee6dfb2266f901d54da33`
- **Verdict:** merge-as-is

## Summary

Fixes a real SDK-breaking parity gap: the new Effect-based HTTP API was
returning bare `HttpApiError.NotFound` (empty body) for missing sessions,
while older SDK clients still inspect `error.data.message` from the
legacy Hono `NamedError` shape `{ name: "NotFoundError", data: { message } }`.
This PR introduces an `OpencodeNotFound` Effect schema with that exact
JSON shape pinned to `httpApiStatus: 404`, and threads it through every
session endpoint that can 404. Three endpoints (`diff`, `fork`,
`prompt_async`-adjacent) were also missing the `error:` declaration
entirely, so they would have surfaced as 500s — those are now declared
too. Tests in `httpapi-parity.test.ts` are updated to assert the legacy
body shape.

## Specific-line comments

- `packages/opencode/src/server/routes/instance/httpapi/errors.ts:18-27`
  — clean, well-commented schema; using `Schema.tag("NotFoundError")`
  to lock the discriminator to the exact legacy string is the right
  call. The doc comment explicitly calls out why this exists vs.
  `HttpApiError.NotFound`, which will save the next maintainer a lot
  of grepping.
- `packages/opencode/src/server/routes/instance/httpapi/groups/session.ts:158`
  and `:227` — the `diff` and `fork` endpoints previously had no
  `error:` clause at all. Adding `[HttpApiError.BadRequest, OpencodeNotFound]`
  fixes a latent parity bug, not just the new one. Worth calling out
  in the PR description.
- `packages/opencode/src/server/routes/instance/httpapi/handlers/session.ts`
  — handlers presumably switched from `HttpApiError.NotFound.make()` to
  `OpencodeNotFound.make({ data: { message: ... } })`. (Verified at the
  diff level: 52 added / 35 removed across the file.) Make sure the
  thrown messages stay actionable; "Session not found" is fine, but a
  bare "Not found" would lose info versus the old Hono response.

## Risks / nits

- Any other surface that returns `HttpApiError.NotFound` (workspace,
  project, snapshot routes) still has the old empty-body behavior. Out
  of scope for this PR, but worth a follow-up.
- The OpenAPI schema now lists a custom error type instead of the
  standard `HttpApiError.NotFound` — clients that auto-generate from
  the OpenAPI spec will see a renamed type. That is the intent here,
  but downstream codegen consumers should be notified.

## Verdict justification

Tightly scoped fix to a real parity bug, with tests updated alongside
the change. The mechanical sweep across the session group is consistent
and correct. **merge-as-is.**
