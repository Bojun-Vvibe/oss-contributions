# Review: google-gemini/gemini-cli #26404 — Stop buffering events when telemetry is disabled

- PR: https://github.com/google-gemini/gemini-cli/pull/26404
- Head SHA: `a65cda0fec699bab26bf79c50799bbb3a707811b`
- Author: genneth
- Files touched: `packages/core/src/config/config.ts`, `packages/core/src/telemetry/sdk.test.ts`, `packages/core/src/telemetry/sdk.ts`

## Verdict: `merge-as-is`

## Rationale

This is a textbook well-scoped memory leak fix. The original bug: `bufferTelemetryEvent` (`sdk.ts:117-126` pre-patch) pushed every `logApi*` closure into `telemetryBuffer` until `telemetryInitialized` flipped true; if telemetry was disabled, init bailed at line 167 *without* draining the buffer, so closures capturing full conversation contents on every `ApiRequestEvent`/`ApiResponseEvent`/`ApiErrorEvent` were retained for the lifetime of the process. The fix introduces a `telemetryDisabled` flag (`sdk.ts:101`) with a clear scope comment distinguishing it from the deferred-init case (waiting on OAuth), short-circuits `bufferTelemetryEvent` at line 126, and in `markTelemetryDisabled()` (`sdk.ts:137`) explicitly drops `telemetryBuffer.length = 0` to free anything queued before init ran. The flag is reset at the top of every `initializeTelemetry` call (`sdk.ts:194`) so config reload / re-enable works; the test at `sdk.test.ts:413-427` covers exactly that. The config.ts change at line 1391 is the matching half — it always calls `initializeTelemetry` even when disabled so `markTelemetryDisabled` actually runs (previously the call was guarded by `if (this.telemetrySettings.enabled)`, which is what allowed the buffer to grow forever). Excellent comments throughout, four new tests covering pre-init buffering, config-conflict path, re-enable-after-disable, and shutdown listener cleanup from the deferred-init state. Ship it.