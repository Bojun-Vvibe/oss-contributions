# google-gemini/gemini-cli #26362 — Guard stdin cleanup after SSH/TTY disconnect

- PR: https://github.com/google-gemini/gemini-cli/pull/26362
- Author: chenjian-agent
- Head SHA: `fc240b010d089d0f3faab49fb565feb901eb2360`
- Updated: 2026-05-02T03:53:50Z

## Summary
Hardens `drainStdin()` in `packages/cli/src/utils/cleanup.ts` so that exit cleanup does not crash when the controlling TTY has already been torn down (e.g. SSH session dropped). Adds early-bailout when `stdin.destroyed | readableEnded | closed | readable === false`, and wraps the resume/listen/wait sequence in a try/catch that swallows any teardown errors. Two regression tests cover the destroyed-stdin path and the throwing-resume path.

## Observations
- `packages/cli/src/utils/cleanup.ts:117-124`: the new guard is correct and reasonably defensive. Note that `stdin.readable === false` check uses strict equality — if Node's typings ever expose `readable` as `undefined` initially, the check still falls through correctly. Good.
- `packages/cli/src/utils/cleanup.ts:128-137`: try/catch around `stdin.resume().removeAllListeners('data').on('data', () => {})` and the 50ms `setTimeout`. The `setTimeout` itself can't throw, so the catch only covers the resume/listener mutation. Acceptable. Consider logging the swallowed error at debug level — silent swallow during shutdown is exactly when telemetry breaks down for debugging future regressions: `debugLogger.debug('drainStdin teardown error', { err })` or similar.
- `packages/cli/src/utils/cleanup.test.ts:127-160`: first test mutates `process.stdin.resume`, `isTTY`, and `destroyed` via direct assignment / `Object.defineProperty`, then restores them in `finally`. Solid pattern. Minor: `process.stdin.resume = resumeSpy` mutates a global; if other tests in the same file run in parallel they would see the spy. Bun and vitest typically serialize within a file, so this is fine, but worth noting.
- `packages/cli/src/utils/cleanup.test.ts:162-186`: second test verifies that a registered `cleanupFn` still runs even when `drainStdin` throws. This is the important behavioral guarantee — verifies error isolation between drain and the rest of `runExitCleanup`. Good.
- The check chain `!stdin?.isTTY || stdin.destroyed || stdin.readableEnded || stdin.closed || stdin.readable === false` is order-sensitive only insofar as short-circuit perf; correct semantics. Could be extracted into a `stdinIsUsable()` helper for readability and so non-TTY callers can reuse the same check.
- No assertion on the *contents* of any swallowed error — fine, but if the logger suggestion above is taken, a test that asserts the debug log is emitted would close the loop.
- 50ms `setTimeout` is unchanged. Pre-existing magic number; out of scope for this PR but worth a TODO/comment about why 50ms.

## Verdict
`merge-after-nits`
