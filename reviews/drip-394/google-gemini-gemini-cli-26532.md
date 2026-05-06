# Review: google-gemini/gemini-cli #26532 — fix(core): reject numeric project IDs in GOOGLE_CLOUD_PROJECT (#24695)

- Head SHA: `54626166336b430f0949e4323532df3717481174`
- Files: `packages/core/src/code_assist/setup.ts`, `packages/core/src/code_assist/setup.test.ts`

## Summary

Closes the long-standing UX trap (issue #24695) where users set
`GOOGLE_CLOUD_PROJECT` (or `_PROJECT_ID`) to their *Project Number* (a
numeric ~10-digit string from the GCP console's "Project info" card) instead
of their string-based *Project ID*. Previously this was forwarded verbatim
to the Code Assist API, which returns an opaque permission error that
gives no hint about the env-var being wrong. Now the auth setup detects an
all-digits project value before the API call and throws a typed
`InvalidNumericProjectIdError` with a message that names both env-var
candidates and an example of the correct shape.

## Specifics

- `packages/core/src/code_assist/setup.ts:39-49` — new
  `InvalidNumericProjectIdError` class extends `Error`, sets
  `this.name = "InvalidNumericProjectIdError"` (correct — `name` set after
  `super()` so `JSON.stringify(err)` shows the right discriminator). Error
  message names both `GOOGLE_CLOUD_PROJECT` and `GOOGLE_CLOUD_PROJECT_ID`
  and gives a concrete `"my-project-123"` example.
- `packages/core/src/code_assist/setup.ts:131-134` — guard at the right
  spot: after the env-var read at `:127-129` (`process.env['GOOGLE_CLOUD_PROJECT']
  || process.env['GOOGLE_CLOUD_PROJECT_ID'] || undefined`) and before the
  `userDataCache.getOrCreate(...)` lookup. The regex
  `/^\d+$/.test(projectId)` is the right check (anchors enforce
  all-digits, no decimal-point or whitespace fall-through).
- `packages/core/src/code_assist/setup.test.ts:11` — adds the new error
  class to the named imports.
- `packages/core/src/code_assist/setup.test.ts:222-235` — two new tests
  pin both env-var paths:
  - `GOOGLE_CLOUD_PROJECT=1234567890` → `rejects.toThrow(InvalidNumericProjectIdError)`
  - `GOOGLE_CLOUD_PROJECT_ID=1234567890` → same

## Nits

- The regex is strict-decimal-only. A user who somehow sets the value to
  `1234567890\n` (trailing newline from a sloppy `export $(cat .env)`)
  will *not* trigger the new error and will get the old opaque API error
  back. A `projectId.trim()` at `:131` (or before the regex) would cover
  that case cheaply. Same for leading/trailing whitespace.
- No test for the numeric-with-leading-zero case (`"0987654321"`). Project
  Numbers don't typically have leading zeros but the regex would correctly
  catch them anyway — adding the test would document intent.
- The new error is thrown but never caught at any visible call site of
  `setupUser` — confirm the surrounding CLI surface formats unknown
  `Error.name` values into a user-readable banner (vs. just stringifying
  `err.message`). If it's the latter, the new error name is wasted; if
  the former, the dedicated class lets that surface render a help link.
- Worth adding a one-line tip to the error message about how to find the
  Project ID (`gcloud projects list` / GCP console > Project Info >
  "Project ID" not "Project number") so users don't have to find the
  associated docs.

## Verdict

`merge-after-nits`
