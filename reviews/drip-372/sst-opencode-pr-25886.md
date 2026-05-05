# sst/opencode#25886 — fix: retry OpenAI overload errors

- **URL:** https://github.com/sst/opencode/pull/25886
- **Head SHA:** `6b8e9fde087f6c2f36bc1dfb66dac9dd259baab3`
- **Closes:** #25884
- **Files touched:** 4 (+76 / −1)
  - `packages/opencode/src/provider/error.ts` (+1 / −0)
  - `packages/opencode/src/session/retry.ts` (+19 / −1)
  - `packages/opencode/test/session/message-v2.test.ts` (+23 / −0)
  - `packages/opencode/test/session/retry.test.ts` (+33 / −0)

## Summary

OpenAI-compatible streams can return `server_is_overloaded` / `service_unavailable_error` as a stream-error event rather than an HTTP error. This PR (a) maps those stream errors into the existing retryable `APIError` path and (b) teaches the generic retry classifier about the same nested provider `code`/`type`/`message` fields whether they arrive as a plain JSON body or as the `responseBody` of an `APIError`.

## Line-level observations

- `packages/opencode/src/provider/error.ts:155` — `case "server_is_overloaded":` falls through to the existing `case "server_error":` arm in `parseStreamError`. That arm already returns `{ type: "api_error", message, statusCode: 503, isRetryable: true, responseBody }`, so no second branch is needed and the new code is just a one-line case label. Clean.
- `packages/opencode/src/session/retry.ts:16-20` — `OVERLOAD_MARKERS` constant is a 3-element string array (`server_is_overloaded`, `service_unavailable_error`, `servers are currently overloaded`). All three are matched lowercased everywhere they're consulted, which is correct (OpenAI emits them lowercased).
- `:65-71` — APIError branch concatenates `error.data.message + " " + error.data.responseBody`, lowercases once, then uses `OVERLOAD_MARKERS.some(...)`. The original `Overloaded` substring check is preserved for the Anthropic capitalized variant. Good defensive layering.
- `:80` — adds `OVERLOAD_MARKERS.some(...)` to the plain-text retry classifier alongside the existing `rate increased too quickly` / `rate limit` / `too many requests` checks. Also good — covers the case where an OpenAI overload error reaches the classifier as plain text rather than parsed JSON.
- `:107-114` — JSON branch reads `code`, `type`, `error.type`, `error.code`, `error.message` and joins them all into one lowercased blob before the marker check. Slight wastefulness vs short-circuiting, but the joined string stays small (<2KB worst case from the test fixtures) and the readability win is real.
- Tests at `test/session/retry.test.ts:129-160` cover both the stream-error path and the `APIError` with `responseBody` path. `test/session/message-v2.test.ts:1183-1203` pins the `fromError` serialization shape end-to-end. Verdict mapping (`"Provider is overloaded"`) is asserted.

## Concerns

- The PR notes overlap with #25728 and offers to close one. Maintainer needs to decide which framing wins; the OpenAI-side framing here is more localized and the provider matrix this fixes (any OpenAI-compatible stream) is broader than just the codex-focused PR.
- `bun` was not available in author's environment, so tests were written but not executed locally. CI will catch any regression.

## Verdict

**merge-after-nits**

## Rationale

Tight, well-tested fix. The only blocker is the maintainer-side decision between this PR and #25728. Once that's resolved, this is mergeable as-is — no code changes requested.
