# google-gemini/gemini-cli#26568 — fix(a2a-server): Resolve race condition in tool completion waiting

- Head SHA: `4f398f17ff350a1d6ea4fd9b89bb59da160fab0e`
- Author: @kschaab
- Link: https://github.com/google-gemini/gemini-cli/pull/26568

## Notes
- `packages/a2a-server/src/agent/race-condition.test.ts:1–172` (new file) reproduces the exact bug: two tools enter `AwaitingApproval`, `waitForPendingTools()` is called, then the first confirmation calls `_resetToolCompletionPromise` which overwrites the promise being awaited. The repro is precise and the test will fail without the source fix.
- The test uses `// @ts-expect-error - private constructor` (line 33) to bypass `Task`'s private ctor — pragmatic for a regression test, but consider exposing a test-only factory to avoid coupling to private internals.
- The mock `messageBus` (lines 21–25) only stubs `subscribe/unsubscribe/publish`; if `Task` later subscribes to additional bus types, the test will silently miss them. Acceptable for a focused race repro.

## Verdict
`merge-after-nits`

Test-only PR that locks down a real concurrency bug. Replace the `@ts-expect-error` with a test-helper constructor before merge, and confirm the underlying source fix is in this PR or a sibling one (the diff excerpt only shows the test).
