# Review: sst/opencode PR #25534

- **Title:** fix: respect retryCount in json_schema structured output
- **Author:** 21pounder
- **Head SHA:** `a6a4d334abc562d97bfab451466fd77749f61e60`
- **Verdict:** merge-after-nits

## Summary

The PR fixes a real bug: when a model returned a non-structured payload
for a `json_schema` output format, the loop immediately surfaced
`StructuredOutputError(retries: 0)` instead of honoring the schema's
`retryCount` (default 2). The fix introduces a `retryAttempt` counter in
the prompt loop and re-runs the step up to `format.retryCount` times
before failing. Schema-side, the two `OutputFormat*` Effect classes are
rewritten as `Schema.Struct` + `withStatics` so the existing `.zod`
helper survives.

## Specific-line comments

- `packages/opencode/src/session/prompt.ts:1402` — adding `retryAttempt`
  at the outer scope (alongside `step`) is correct: a schema retry is
  conceptually the same loop iteration repeated, so resetting it per
  step would defeat the cap. Fine as-is.
- `packages/opencode/src/session/prompt.ts:1597-1610` — the new branch
  reads `format.retryCount ?? 2` on every finished message. Two nits:
  (a) the default is duplicated against the schema default
  (`withDecodingDefault(... succeed(2))` in `message-v2.ts:65`); pull a
  named constant or trust the decoded default to avoid drift; (b) when
  the cap is exhausted, the error reports `retries: retryAttempt` which
  is the number of *retries already attempted*, not "attempts remaining"
  — that matches the field name but worth a one-line comment so a
  future reader does not flip it.
- `packages/opencode/src/session/message-v2.ts:62-78` — converting from
  `Schema.Class` to `Schema.Struct` + `withStatics` keeps the runtime
  shape but loses the nominal-typing Effect classes give you. The
  exported `type OutputFormatJsonSchema = Schema.Schema.Type<typeof …>`
  is structural; any consumer that previously did
  `instanceof OutputFormatJsonSchema` will silently break. Worth a quick
  grep before merge.

## Risks / nits

- No new test exercises the retry path. A unit test that drives the
  prompt loop with a mock model returning two unstructured responses
  followed by a valid one would lock in the fix.
- The `return "continue"` path bypasses the `handle.message.error`
  assignment but does it also reset the partial structured payload
  buffered into `structured` from the prior attempt? Worth verifying
  that a half-parsed object from attempt N does not leak into the
  decode-success check of attempt N+1.

## Verdict justification

The behavior change is correct and the diff is small (17/9). The two
nits (named constant + an `instanceof` audit) are minor and not blocking,
but a regression test would meaningfully strengthen this. **merge-after-nits.**
