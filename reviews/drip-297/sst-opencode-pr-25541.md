# Review: sst/opencode PR #25541

- **Title:** fix: add error msg if well known config url cannot be fetched
- **Author:** rekram1-node (Aiden Cline)
- **Head SHA:** `b417e1eb9df5f01bb0d509affb22a340534352ad`
- **Verdict:** merge-after-nits

## Summary

One-line change in `packages/opencode/src/config/config.ts:501`. The
remote-config bootstrap previously did
`yield* Effect.promise(() => fetch(...))`, which treats network failures
as defects (uncaught). On a DNS error or unreachable wellknown host the
program would die without a useful message. This PR switches to
`Effect.tryPromise(...).pipe(Effect.mapError(cause => new Error(...)))`
so the failure becomes a typed error with the offending URL embedded.

## Specific-line comments

- `packages/opencode/src/config/config.ts:501` — correct fix shape:
  `Effect.promise` is for infallible promises, `Effect.tryPromise` is
  the right primitive when `fetch` can reject. The `mapError` rewrap to
  a plain `Error(...)` keeps the error channel as `Error` so existing
  call sites continue to work without a type-channel widening.
- The error message format is slightly off from the line below it:
  this PR uses `failed to fetch remote config from wellknown provider
  ${url}: ${cause}` while line 503 uses `failed to fetch remote config
  from ${url}: ${response.status}`. The "wellknown provider" wording is
  a minor inconsistency — pick one phrasing.
- `${cause}` will stringify a `FetchError` / `TypeError` reasonably on
  Node 22+, but on Bun the underlying error sometimes has `cause.cause`
  with the actually useful info (e.g. `ECONNREFUSED`). Consider
  `cause instanceof Error ? cause.message : String(cause)` or
  including `cause.cause` when present.

## Risks / nits

- No test added. A small Effect test that swaps `fetch` for a rejecting
  stub and asserts the error message contains the URL would lock this
  in.
- The mapped error throws away the original `cause` chain. Logging
  consumers downstream lose the stack. Using `new Error(msg, { cause })`
  preserves it on Node 18+/Bun.

## Verdict justification

Trivially correct one-line fix that converts a defect into a typed
error with actionable context. The wording inconsistency with the
adjacent error and the dropped `cause` chain are nits worth fixing
before merge but don't block. **merge-after-nits.**
