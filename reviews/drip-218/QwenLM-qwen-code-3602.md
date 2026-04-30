---
pr-url: https://github.com/QwenLM/qwen-code/pull/3602
sha: 895e2ce989e2
verdict: merge-after-nits
---

# fix(cli): drain runExitCleanup before process.exit in error handlers

Closes a real shutdown-ordering bug in the non-interactive CLI: error-path handlers (`handleError`, `handleCancellationError`, `handleMaxTurnsExceededError`, `handleToolError`) previously called `process.exit(...)` synchronously, racing the registered cleanup callbacks (telemetry flush, file handles, child-process kills). The fix at `packages/cli/src/utils/errors.ts` makes those handlers `async` and `await`s a `runExitCleanup()` drain before exiting; every call site at `packages/cli/src/nonInteractiveCli.ts:349-498,711,766` is correspondingly converted from `handleError(error, config)` to `await handleError(error, config)`.

The interesting test-side change is the introduction of `_resetCleanupFunctionsForTest()` and `_resetExitLatchForTest()` exports plus paired `beforeEach` calls at `nonInteractiveCli.test.ts:81-87` and `errors.test.ts:99-103`. The comment at the first reset site is unusually candid and worth quoting: "Without these resets the once-set exit latch parks subsequent JSON-mode handleError tests in the never-resolving promise (5s vitest timeout)." That tells us the latch is module-level singleton state — the right fix would be to scope it per-`Config` instance, but exposing test-only resetters is a defensible pragmatic step.

The test-shape change at `errors.test.ts:179-186` from `expect(() => handleError(...)).toThrow(testError)` to `await expect(handleError(...)).rejects.toThrow(testError)` correctly threads through the `async` boundary.

Nits: (1) `_resetExitLatchForTest` and `_resetCleanupFunctionsForTest` use the underscore-prefix convention but are real exports — a `__TEST_ONLY__` namespace or a build-time `if (process.env.NODE_ENV !== 'test') throw` guard would harden against accidental production calls; (2) the title says "drain runExitCleanup before process.exit in error handlers" but the change also widens `handleMaxTurnsExceededError` and `handleCancellationError` from sync to async, which is a contract change for any third-party caller — needs a CHANGELOG note; (3) no test pins the *ordering* claim (cleanup-then-exit) — only that the function rejects/throws; a spy that asserts `runExitCleanup` was awaited before the latched exit would lock the actual fix.

## what I learned
Module-level singleton latches that gate `process.exit` are a recurring shutdown-bug source — the right shape is to scope the latch to the `Config` instance so test isolation is automatic, but where that's a bigger refactor, candid `_resetForTest` exports with explanatory comments are the honest interim fix.
