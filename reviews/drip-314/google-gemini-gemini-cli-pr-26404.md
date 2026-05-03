# Review — google-gemini/gemini-cli #26404: fix(telemetry): stop buffering events when telemetry is disabled

- **PR:** https://github.com/google-gemini/gemini-cli/pull/26404
- **Head SHA:** `a65cda0fec699bab26bf79c50799bbb3a707811b`
- **Author:** genneth
- **Files:** 3 changed (+152 / −40)
  - `packages/core/src/telemetry/sdk.ts` (+70/−36)
  - `packages/core/src/telemetry/sdk.test.ts` (+72)
  - `packages/core/src/config/config.ts` (+10/−4)

## Verdict: `merge-as-is`

## Rationale

This is a real memory-leak fix with a clear root cause and a tight reproduction. Previously, `Config` skipped calling `initializeTelemetry(this)` when `telemetrySettings.enabled` was false (`config.ts:1391-1395` of the old code). But the rest of the codebase still calls `bufferTelemetryEvent(fn)` on every API request/response/error event, and `bufferTelemetryEvent` would push the closure onto a module-level `telemetryBuffer[]` whenever `telemetryInitialized` was `false`. Net effect: with telemetry disabled, every API event closure (which captures the full conversation contents on `ApiRequestEvent` / `ApiResponseEvent` / `ApiErrorEvent`) was retained for the lifetime of the process and grew unbounded. The PR description confirms multi-GB retention live in long sessions with frequent API errors.

The fix has the right shape in three parts. First, `config.ts:1392-1401` always calls `initializeTelemetry(this)` — the comment makes the rationale explicit. Second, `sdk.ts:101-109` introduces a new `telemetryDisabled` flag that's *separate* from `telemetryInitialized`, and `bufferTelemetryEvent` at `sdk.ts:124-127` short-circuits early when it's set. Third, `markTelemetryDisabled()` at `sdk.ts:137-143` clears `telemetryBuffer.length = 0` so any events that buffered before init runs are also dropped — without this, the leak would still happen for events submitted between module load and the first init call.

Critically, the deferred-init case (waiting for OAuth credentials) is preserved correctly: the comment at `sdk.ts:104-108` explicitly notes that path "intentionally leaves this false so events still buffer and get flushed once init completes". The reset at `sdk.ts:198-204` ("Reset the disabled flag in case a previous init had set it") handles the re-init case where config is reloaded. The cleanup of the auth listener in `shutdownTelemetry` (the trailing diff hunk around `sdk.ts:447-453`) closes a separate latent listener leak that the new tests exercise.

Test coverage is excellent: regression test for the disabled-init case (`sdk.test.ts:362-385`), the `useCollector + useCliAuth` conflict path (`sdk.test.ts:387-398`), the re-enable-after-disable scenario (`sdk.test.ts:400-415`), and the listener-cleanup case (`sdk.test.ts:417-434`). All four are scenarios you'd plausibly regress. Merge as-is.

## Banned-string check

Diff scanned; no banned tokens present.
