# Review: anomalyco/opencode #25980 — fix(desktop): suppress EPIPE errors in console transport

- Head SHA: `3683173af343221f65fac9002696aae7103d0daa`
- Diff size: +19 / -0 across 1 file
- File: `packages/desktop/src/main/logging.ts`

## Summary

Wraps the electron-log console transport's `writeFn` so that an `EPIPE` from the
underlying stream (typical when the parent terminal/stdout closes ahead of the
Electron main process — common on Windows when the launcher pipe goes away) is
caught instead of bubbling up as an uncaught exception that kills the main
process. On `EPIPE`, the console transport is permanently silenced
(`level = false`); any other error is rethrown.

## Specifics

- `logging.ts:9` — `initConsoleTransport()` is wired into `initLogging()` before
  `cleanup()`; ordering looks correct (you want the wrapped `writeFn` installed
  before any subsequent log line).
- `logging.ts:18-27` — `write = log.transports.console.writeFn.bind(...)`
  captures the original transport once; the new `writeFn` delegates and
  swallows only `EPIPE`. Good: no infinite recursion risk because the bound
  reference is captured before the closure is installed.
- `logging.ts:30-32` — `isBrokenPipe` is a tight type-narrowed guard
  (`"code" in err && err.code === "EPIPE"`). Will not match aggregated
  errors (`AggregateError` from a multi-write fallthrough), but those are
  not produced by electron-log's stdout sink in practice.

## Nits

- The `level = false` mutation is silent — a single-line `log.warn(...)` (to
  the file transport, which is unaffected) before disabling would make the
  silencing observable in support bundles. Right now a user reporting
  "console quiet after some time" gives operators nothing to grep for.
- No regression test. A unit test that stubs `writeFn` to throw a synthesized
  `{ code: "EPIPE" }` and asserts (a) no rethrow and (b)
  `log.transports.console.level === false` afterwards would pin the contract.
- Consider also catching `ERR_STREAM_DESTROYED` here — same root cause class
  on Node 22+ when the writable side has been torn down out from under the
  transport.

## Verdict

`merge-after-nits`
