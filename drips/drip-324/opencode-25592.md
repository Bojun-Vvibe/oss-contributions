# sst/opencode #25592 — fix(sdk+cli): surface real errors instead of bare {} when server returns empty body

- Link: https://github.com/sst/opencode/pull/25592
- Head SHA: `0ea661d5a22c2325aeff02f30af7eed0aecb1b5c`
- State: MERGED 2026-05-03, +49/-10 across 3 files

## Summary
Two related fixes for the long-standing "the CLI just printed `{}` and I have
no idea what happened" problem:

1. `errorFormat()` no longer returns the literal string `"{}"` for plain
   objects whose own properties are all non-enumerable (or empty). It falls
   back through `String(error)` → constructor name + `getOwnPropertyNames()`
   → `"<Ctor> (no message)"`.
2. The generated v2 SDK client previously threw `{}` when the server replied
   with an empty/unparseable error body. A new `client.interceptors.error`
   wraps **only** that case in a real `Error` carrying method, URL, and
   status, while letting any parsed JSON error body through unchanged.

The dead helper `errorFormatted()` is removed and its three call sites in
`errorData()` now call `errorFormat()` directly.

## Specific references
- `packages/opencode/src/util/error.ts:10-22` — new `if (json === "{}")`
  fallback chain producing `<Ctor> (no message)` /
  `<Ctor> { propA, propB }`.
- `packages/opencode/src/util/error.ts:49` — `errorMessage` no longer needs
  the `formatted !== "{}"` guard since `errorFormat` no longer returns `{}`.
- `packages/opencode/src/util/error.ts:60,68,86` — `errorFormatted(error)`
  → `errorFormat(error)` (helper deleted).
- `packages/opencode/test/util/error.test.ts:25-35` — new test pins
  `errorFormat({}) !== "{}"` and `errorFormat(opaque).toContain("OpaqueError")`.
- `packages/sdk/js/src/v2/client.ts:88-104` — adds the
  `client.interceptors.error.use(...)` that wraps empty errors with a
  diagnostic `Error(...)`.

## Notes
- Conservative interceptor: it only rewrites `undefined`/`null`/`""`/empty
  plain object cases and explicitly preserves real `Error` instances
  (`!(error instanceof Error)`), so existing consumers that inspect parsed
  JSON error bodies are unaffected.
- The interceptor's diagnostic message is bounded (method/url/status)
  and won't accidentally leak request bodies — good privacy hygiene.
- Already merged; this is a post-hoc review for the dataset.

## Verdict
`merge-as-is`
