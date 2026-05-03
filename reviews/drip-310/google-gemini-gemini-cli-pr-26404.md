# google-gemini/gemini-cli #26404 — fix(telemetry): stop buffering events when telemetry is disabled

- PR: https://github.com/google-gemini/gemini-cli/pull/26404
- Author: genneth
- Head SHA: `a65cda0fec699bab26bf79c50799bbb3a707811b`
- Updated: 2026-05-03T14:11:29Z
- Fixes: #21255

## Summary
`telemetryBuffer` in `packages/core/src/telemetry/sdk.ts` grew unbounded when telemetry was disabled — `bufferTelemetryEvent` would push closures capturing `ApiResponseEvent` / `ApiErrorEvent` payloads (which retain the full conversation contents) but `flushTelemetryBuffer` early-returned because `telemetryInitialized` stayed `false` on the disabled path. Author reports a live debug session retaining ~2.5 GB heap / 56 copies of a 48 MB request body. PR adds a `telemetryDisabled` flag, marks it on every permanent-off branch, drops anything already buffered, and short-circuits subsequent pushes. Also fixes a sibling listener-leak in `shutdownTelemetry` for the deferred-init (CLI auth pending) state.

## Observations
- `packages/core/src/config/config.ts:1389-1401`: removing the `if (this.telemetrySettings.enabled)` guard around `initializeTelemetry(this)` is the right move — the function now reliably flips `telemetryDisabled` for any caller, and the inline comment explains why. Good.
- `packages/core/src/telemetry/sdk.ts:101-108`: `telemetryDisabled` declaration includes a comment that explicitly distinguishes the *deferred-init* case (CLI auth waiting on credentials) from the permanent-off case. That distinction is the whole correctness story of this PR; keep that comment intact through any future cleanup.
- `packages/core/src/telemetry/sdk.ts:126-128`: `bufferTelemetryEvent` now short-circuits on `telemetryDisabled` *before* the `telemetryInitialized` check. Order matters here — since deferred-init leaves both flags `false`, the deferred-init case still buffers as designed. Good.
- `packages/core/src/telemetry/sdk.ts:137-144`: `markTelemetryDisabled()` zeroes `telemetryBuffer.length` to drop closures captured before init ran. Note: `telemetryBuffer.length = 0` is the right way (vs reassigning `telemetryBuffer = []`) because `bufferTelemetryEvent` and `flushTelemetryBuffer` both close over the array reference. Verify there is no other module that imports the array directly — a quick `grep` for `telemetryBuffer` should confirm. From the diff, only `sdk.ts` references it, so this is safe.
- `packages/core/src/telemetry/sdk.ts:189-200`: the *re-init* reset `telemetryDisabled = false` after the first early-return guard is subtle but correct — it lets a config reload flip telemetry back on without forcing a process restart. The unit test `should re-enable buffering after a disabled init when telemetry is later enabled` covers this exact flip; good.
- `packages/core/src/telemetry/sdk.ts` `shutdownTelemetry`: the rewrite splits "tear down OTel SDK" (gated on `telemetryInitialized && sdk`) from "always reset module-level state and detach listeners". That's the right shape — the original `try/finally` only ran the listener cleanup if the SDK had actually started, which is what leaked `authListener` in the deferred-init case. The new `should clean up the auth listener when shutdown is called from the deferred-init state` test pins this. Good.
- `telemetryBuffer.length = 0` is also added to `shutdownTelemetry` — that's defensive and correct; otherwise a re-init after shutdown could replay stale closures.
- Tests at `packages/core/src/telemetry/sdk.test.ts:359-432`: three new regression tests cover (a) disabled-telemetry path drops both pre- and post-init events, (b) `useCollector + useCliAuth` conflict path, (c) re-init flip back on. Plus the listener-leak test. Coverage is thorough for the affected code.
- One small concern: the deferred-init comment promises that *the deferred-init case (waiting for OAuth credentials) intentionally leaves this false so events still buffer and get flushed once init completes*. There is no new test for that case. Worth one more test that asserts events buffered between the first deferred init and the post-auth re-init are actually flushed — otherwise a future change to the deferred branch could silently start dropping events and existing tests wouldn't catch it.
- PR validation matrix only checks Windows + npm. macOS / Linux validation would be nice for a memory-leak fix that affects every long-running session, but the change is platform-agnostic so this is a soft ask.

## Verdict
`merge-after-nits`
