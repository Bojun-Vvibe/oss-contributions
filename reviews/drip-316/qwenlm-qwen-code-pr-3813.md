# Review: QwenLM/qwen-code #3813 — fix(telemetry): add bounded shutdown timeout and fix service.version resource attribute

- PR: https://github.com/QwenLM/qwen-code/pull/3813
- Head SHA: `d8dbdbd8d17e1bbef92641d2920c53699460eb51`
- Author: doudouOUC (jinye)
- Size: +59 / -3

## Summary
Two small but well-scoped telemetry fixes:
1. `service.version` was being set to `process.version` (the Node.js
   runtime version) instead of the CLI's own version, making
   per-release filtering in observability tooling impossible.
2. `shutdownTelemetry()` awaited `currentSdk.shutdown()` with no
   bound, so a hung exporter could prevent process exit.

## Specific citations
- `packages/core/src/telemetry/sdk.ts:138-139` — replaces
  `[SemanticResourceAttributes.SERVICE_VERSION]: process.version`
  with `config.getCliVersion() || 'unknown'`. Correct fix; the
  `'unknown'` fallback prevents `undefined` from becoming a
  meaningless attribute value.
- `packages/core/src/telemetry/sdk.ts:70` — adds
  `const SHUTDOWN_TIMEOUT_MS = 10_000;` as a module-level constant.
  10s is a sensible default for "we are exiting anyway".
- `packages/core/src/telemetry/sdk.ts:285-308` — the timeout
  implementation uses `Promise.race([sdkShutdown, timeout])` with
  `setTimeout(..., 10_000).unref?.()` so the timer doesn't itself
  block exit if the race already resolved. Good detail.
- `packages/core/src/telemetry/sdk.ts:289` — `sdkShutdown.catch(()
  => {})` to swallow the unhandled rejection if the SDK rejects
  *after* the timeout already won the race. Necessary; otherwise
  Node would log a warning at exit. Solid.
- `packages/core/src/telemetry/sdk.ts:298-301` — on timeout, both
  `diag.warn` and `debugLogger.warn` get the message. Slightly
  redundant but not harmful — `diag` goes to the OTel internal
  logger, `debugLogger` to the CLI's own debug stream.
- `packages/core/src/telemetry/sdk.ts:307-309` — `clearTimeout(timer)`
  in the catch block too — prevents a leaked timer if `currentSdk.
  shutdown()` throws synchronously. Defensive, correct.
- `packages/core/src/telemetry/sdk.test.ts:115` — adds
  `getCliVersion: () => '1.0.0-test'` to the mock config so the
  service-version test has something deterministic to assert.
- `packages/core/src/telemetry/sdk.test.ts:336-345` — direct
  assertion that
  `resource.attributes['service.version']` equals `'1.0.0-test'`
  AND `not.toBe(process.version)`. Good defensive double-check.
- `packages/core/src/telemetry/sdk.test.ts:347-369` — fake-timer
  test for the hung-shutdown path: `NodeSDK.prototype.shutdown` is
  stubbed to a never-resolving promise, then `vi.
  advanceTimersByTimeAsync(10_000)` is awaited and
  `isTelemetrySdkInitialized()` is asserted false. Tight.

## Verdict
**merge-as-is**

Small, focused, well-tested. Both fixes address real production
pain points. The implementation handles the edge cases (post-race
rejection, sync throws, timer cleanup) correctly.
