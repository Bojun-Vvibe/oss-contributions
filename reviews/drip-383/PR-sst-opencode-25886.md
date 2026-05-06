# sst/opencode#25886 — fix: retry OpenAI overload errors

- **Head SHA**: `6b8e9fde087f6c2f36bc1dfb66dac9dd259baab3`
- **Author**: johnwaldo
- **Stats**: +76 / -1, 4 files (closes #25884; overlap noted with #25728)

## Summary

Maps OpenAI's `service_unavailable_error` / `server_is_overloaded` stream chunks to the existing retryable API-error path. Three classifier branches (`parseStreamError`, `APIError` body inspection in `retryable`, plain-text/JSON heuristics in `retryable`) all gain a shared `OVERLOAD_MARKERS` const so the same vocabulary triggers retry regardless of error shape. Adds 2 unit tests and 1 stream-error parsing test.

## Specific citations

- `provider/error.ts:155` adds `case "server_is_overloaded":` falling through to the existing `server_error` → `api_error` branch — minimal, correct.
- `session/retry.ts:16-20` introduces `OVERLOAD_MARKERS = ["server_is_overloaded", "service_unavailable_error", "servers are currently overloaded"]`. Lowercase match by construction.
- `retry.ts:68-72` (in HEAD `6b8e9f`): `const text = [error.data.message, error.data.responseBody].filter(Boolean).join(" ").toLowerCase()` — correctly inspects both top-level message *and* responseBody for the markers (this is the load-bearing fix; the previous code only checked `message.includes("Overloaded")`).
- `retry.ts:80` extends the plain-text retryable heuristic with the same markers — covers the case where the SDK surfaces the error as a generic string rather than parsed JSON.
- `retry.ts:107-113` extends the JSON-shape branch: walks `code` + `type` + `error.type` + `error.code` + `error.message`, joins lowercased, runs `OVERLOAD_MARKERS.some(...)`. Wide net but appropriate for streaming chunks where the error envelope shape varies by provider.
- Tests at `retry.test.ts:129-160` cover both the wrapped-stream-chunk shape and the `APIError` `responseBody` shape; `message-v2.test.ts:1183-1204` pins the parser side returning `isRetryable: true`.

## Verdict

**merge-after-nits**

## Rationale

This is a real bug — OpenAI's response API absolutely does emit `service_unavailable_error` as a stream-chunk type and the previous classifier silently dropped it. The 3-branch coverage is thorough and idiomatic. Nits: (1) `OVERLOAD_MARKERS` is module-private but should probably be exported for #25728 to share rather than duplicate; (2) the third marker `"servers are currently overloaded"` is a literal English substring of the OpenAI message and risks i18n drift if OpenAI ever localizes — a regex on `code === "server_is_overloaded"` would be more robust than substring match; (3) PR overlap with #25728 should be reconciled by maintainer before merge — author already flagged this.

