# QwenLM/qwen-code PR #3798 — Classify retryable transport/provider failures vs deterministic request errors

- Head SHA: `4606f563d75551fac6c458647c21ea3f75627728`
- URL: https://github.com/QwenLM/qwen-code/pull/3798
- Size: +396 / -4, 2 files (`packages/core/src/utils/retry.ts`,
  `packages/core/src/utils/retry.test.ts`)
- Verdict: **merge-after-nits**

## What changes

Replaces the previous `defaultShouldRetry` which only retried on
`status === 429 || (status >= 500 && status < 600)` with a richer
classifier that distinguishes:

- Deterministic request errors (400, 401, 403, 404, 422) — never retry.
- Retryable status codes (408, 429, 5xx).
- 409 Conflict — retryable iff the message mentions
  `lock`/`conflict`/`contention`/`409` (`isTransientConflict`,
  retry.ts:316-324).
- Network transport errors via Node `errno` codes (`ECONNRESET`,
  `ETIMEDOUT`, `ESOCKETTIMEDOUT`, `ECONNREFUSED`, `ENOTFOUND`,
  `EHOSTUNREACH`, `EAI_AGAIN`) and message-substring fallbacks
  (`socket closed`, `stream ended`, `network error`, `connection reset`,
  `econnreset`, `etimedout`).

Adds an exported `classifyError(err): { retryable, reason, status? }`
for callers that want the *reason* (logging / telemetry), not just a
boolean.

## What looks good

- `defaultShouldRetry` now reads as a clean `if/else` ladder
  (retry.ts:265-289) instead of one mystery boolean expression.
  Easier to extend and easier to grep when an SRE asks "why did this
  retry."
- Splitting `isRetryableNetworkError` into its own export
  (retry.ts:330-353) means the streaming path can call it directly
  without going through the full `defaultShouldRetry` machinery.
- The 14 `classifyError` cases (test.ts:110-243) cover every status
  branch including the trickiest (`409 with conflict message` →
  retryable, `409 without` → non-retryable). Test file is the spec.
- `RETRYABLE_NETWORK_CODES` as a `Set` (retry.ts:302-310) — both
  faster than an array `.includes` and the right intent (membership,
  not order).

## Nits

1. `isTransientConflict` (retry.ts:316-324) does
   `message.includes('409')`, which means the *literal substring* "409"
   in any error message — including a 4xx error from a provider that
   prefixes status codes into the message body ("HTTP 409: ...") *and*
   a 9xx-something edge case ("error code 4091"). Drop the `'409'`
   substring check; it's redundant (the caller already checked
   `status === 409`) and produces false positives.
2. The message-substring matchers (retry.ts:341-348:
   `socket closed`, `stream ended`, `network error`, `connection
   reset`, `econnreset`, `etimedout`) are case-insensitive via
   `.toLowerCase()`, but a provider that returns
   `{ "error": "Socket closed by peer", "code": "ABORTED" }` with a
   `code` field that *isn't* `ECONNRESET` will be classified as a
   retryable network error purely on text. That's probably fine, but
   note that this ladders order-of-precedence: the `isNodeError`
   branch (retry.ts:332-337) only matches if `error.code` is in the
   exact-7 set; everything else falls to substring matching. Add a
   single test case where `error.code = 'EACCES'` *and* `error.message
   = 'connection reset'` to pin which one wins (the message-based
   match wins, per current code — verify that's intentional).
3. `classifyError` returns `reason` strings that downstream callers
   are likely to log or telemetry. Those strings are unstructured
   ("Server error (HTTP 500)", "Rate limited (HTTP 429)") — fine for
   logs, but consider also returning a stable `kind:
   "deterministic_request" | "rate_limited" | "timeout" |
   "transient_conflict" | "deterministic_conflict" | "server_error" |
   "network" | "unknown"` discriminator so dashboards don't have to
   parse English.
4. No test for `409` *without* a `message` field (the
   `error = { status: 409 }` shape — `getErrorMessage` will return
   something like `"undefined"` or `""`, hitting the
   `Deterministic conflict` branch). Worth pinning.
5. The PR adds 7 imports from `./errors.js` (retry.ts:254-259) but
   `isNodeError` is the only one not previously imported.
   `getErrorMessage` and `getErrorType` are new in this file — verify
   they exist in `errors.js` (the diff doesn't show them); a missing
   export here would only fail at the test step, but a `tsc --noEmit`
   would catch it earlier.

## Risk

Medium. The previous behavior was conservative ("retry only 429 and
5xx"); the new behavior retries strictly more often (now also 408,
some 409s, network errnos, and a fairly wide set of message
substrings). For a well-behaved provider this is a clear improvement,
but for one that returns "stream ended" as a *user-input rejection*
(unlikely but possible), this PR will now retry that into the dust
and consume the user's quota. Worth one observability follow-up: emit
a `classification.reason` counter so a regression here would show up
as a sudden spike in `network` retries.
