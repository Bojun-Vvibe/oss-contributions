# sst/opencode#25728 — fix(session): retry Codex server_is_overloaded stream errors

- PR ref: `sst/opencode#25728` (https://github.com/sst/opencode/pull/25728)
- Head SHA: `ae3860b2110fa3ce37b8fc375a7bb25fe8de2d5d`
- Title: fix(session): retry Codex server_is_overloaded stream errors
- Verdict: **merge-as-is**

## Review

Three changes that all hit the right spot. First, the new `case "server_is_overloaded"`
arm at `packages/opencode/src/provider/error.ts:155` falls through into the existing
`server_error` branch, so the chunk gets reified as an `api_error` with the correct
`responseBody` and `status` rather than being dropped on the floor. That's the minimum
necessary to make `MessageV2.fromError` emit a retryable `APIError`, which is exactly
what the new test at `test/session/retry.test.ts:339-362` proves end-to-end.

Second, the `.toLowerCase()` normalization at `retry.ts:63` is an obvious bug fix —
upstream providers send `"overloaded"` in mixed case in the wild and the previous
substring check was case-sensitive. Third, the `codes` array at `retry.ts:91-95` now
inspects `json.code`, `json.error.code`, **and** `json.error.type`, which is the right
generalization given that Anthropic-shape errors put the code under `error.code` while
OpenAI-shape errors sometimes put it under `error.type`. Adding `"overloaded"` to the
matched substring set at `retry.ts:100` and folding `"too_many_requests"` into the
rate-limit branch at `retry.ts:106` are both well-scoped.

The new test at `retry.test.ts:129-141` uses a realistic chunk shape (Anthropic stream
error envelope with `sequence_number` and nested `error.code`), and the second test
covers the full pipeline through `MessageV2.fromError`. No nits.
