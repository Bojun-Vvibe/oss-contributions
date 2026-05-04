# sst/opencode #25717 — fix: ensure effect server middleware properly parses errors

- **Head SHA:** `803e01f377ae0fdce79144cda2ca538f7bb3b581`
- **Size:** +59 / -0 across 2 files
- **Verdict:** **merge-after-nits**

## Summary
Adds a new `errorLayer` HTTP middleware
(`packages/opencode/src/server/routes/instance/httpapi/middleware/error.ts`)
that catches Effect *defects* (`Cause.hasDies`) instead of letting them surface
as empty 500s, then maps known error classes to proper HTTP status codes:
`NotFoundError → 404`, `Provider.ModelNotFoundError → 400`,
`ProviderAuthValidationFailed → 400`, `Worktree*` errors → 400,
`Session.BusyError → 400`, anything else → 500 with stack/message in the body.
Wired into the Effect router via `Layer.provide([errorLayer, …])` in
`server.ts:148`.

## Strengths
- The boundary is correctly scoped via `Cause.hasDies` (error.ts:16) so typed
  HttpApi failures keep flowing through their declared error path; this is
  exactly the right shape for an Effect server, and the comment on line 12
  explicitly calls it out.
- `HttpServerResponse.isHttpServerResponse(error)` and
  `HttpServerError.isHttpServerError(error)` short-circuits (lines 19-22) avoid
  double-wrapping responses that the framework already produced — a common
  mistake in defect handlers.
- The `error.name.startsWith("Worktree")` mapping (line 32) is a pragmatic way
  to bucket the worktree family without enumerating every subclass; reasonable
  given that worktree errors are virtually always client-driven 4xx.

## Nits
- `error.name.startsWith("Worktree")` (error.ts:32) is a stringly-typed dispatch
  that will silently lose the 400 mapping if a class is renamed. A small
  `WorktreeError` base class with `instanceof` would be more refactor-safe.
- The fallback branch (lines 44-51) returns `error.stack` in the response body
  for any non-`NamedError` defect. That leaks server file paths to whatever
  client hit the endpoint. Worth gating behind a dev-only flag, or returning
  `error.message` to clients while logging `error.stack` server-side
  (`log.error` already runs at line 24).
- `errorLayer` only catches *die* causes. If a handler raises a typed failure
  that is not declared on the HttpApi schema (uncommon but possible when
  refactoring), it will still escape. A short comment noting this would help
  future maintainers — the existing comment covers the inverse case but not
  this one.
- No tests in this PR. Given this layer changes the response shape for every
  unhandled defect, at minimum a couple of tests asserting that `NotFoundError`
  → 404 and an arbitrary `throw new Error(...)` → 500 with no stack in the body
  would be cheap insurance.

## Recommendation
Land after redacting `error.stack` from the 500 response body (or guarding it
behind a debug flag) and adding minimal middleware tests for the
`NamedError` → status mapping.
