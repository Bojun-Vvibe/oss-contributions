# google-gemini/gemini-cli PR #26362 — Guard stdin cleanup after SSH/TTY disconnect

- **Head SHA:** `fc240b010d089d0f3faab49fb565feb901eb2360`
- **Files:** 2 (`packages/cli/src/utils/cleanup.ts`, `packages/cli/src/utils/cleanup.test.ts`)
- **LOC:** +84 / −9

## Observations

- `cleanup.ts:117-126` — the new precondition check expands `!process.stdin?.isTTY` to also bail when `stdin.destroyed`, `stdin.readableEnded`, `stdin.closed`, or `stdin.readable === false`. This is the correct guard: after an SSH disconnect, `isTTY` can still report `true` (Node caches it at process start) while the underlying file descriptor has been torn down, and `.resume()` on a destroyed stream throws. Each predicate is individually justified — `destroyed` for an explicit `.destroy()` call, `readableEnded` for a clean EOF, `closed` for a closed handle, and `readable === false` for a stream that was never readable.
- `cleanup.ts:128-139` — wrapping the `resume().removeAllListeners('data').on('data', ...)` chain plus the 50ms `setTimeout` in `try { ... } catch {}` is the right defense. The catch is intentionally empty with a clarifying comment that this runs during exit and stdio teardown errors must not block other cleanup. Acceptable swallow given the comment.
- `cleanup.test.ts:129-159` — `should skip draining stdin when tty input has already been torn down` properly stubs `isTTY=true, destroyed=true`, asserts `resumeSpy` is not called, and restores the originals in `finally`. Good test hygiene; no leakage between tests.
- `cleanup.test.ts:161-186` — `should ignore errors raised while draining stdin during shutdown` registers a normal cleanup fn, makes `resume()` throw, and asserts `runExitCleanup()` resolves without throwing AND that the registered cleanup ran. This is the key behavioral guarantee: a stdio failure must not abort the rest of the cleanup pipeline.
- The test mocks mutate `process.stdin` global state directly via `Object.defineProperty`. This is fragile under parallel test runners — confirm vitest is configured to run this file serially or that the suite-level `beforeEach`/`afterEach` (not shown in diff) restores defaults. The `try/finally` per-test does mitigate this but doesn't cover the case where the test crashes between `defineProperty` and `try`.

## Verdict: `merge-after-nits`

- Confirm vitest isolation for `cleanup.test.ts` — process.stdin mutation in shared global state can leak across files.
- The fix itself is small, targeted, and correct.
