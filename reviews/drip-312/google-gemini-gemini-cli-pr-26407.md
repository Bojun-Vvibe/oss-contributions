# google-gemini/gemini-cli PR #26407 — fix: await IDE client initialization to prevent race condition

- URL: https://github.com/google-gemini/gemini-cli/pull/26407
- Head SHA: `26c9e4fc6725ae52586c43ca6e5afbf45d53fda7`
- Verdict: **merge-after-nits**

## Summary

Converts the previously fire-and-forget IDE client bootstrap inside
`initializeApp` into an awaited `try/catch` block, so that
`initializeApp` only resolves once `IdeClient.getInstance()` and
`ideClient.connect()` have finished (or thrown). The accompanying test
asserts that `initializeApp` does not resolve until the connect promise
resolves.

## Specific references

- `packages/cli/src/core/initializer.ts:56-68` — the substantive change.
  The previous code path was:

  ```ts
  IdeClient.getInstance()
    .then(async (ideClient) => { await ideClient.connect(); … })
    .catch((e) => { debugLogger.error(…) });
  ```

  i.e., a detached promise chain. The new code awaits sequentially
  inside a `try/catch`, so the function only `return`s after IDE
  state is known.

- `packages/cli/src/core/initializer.test.ts:101-130` — rewritten test.
  Constructs a manually-controlled `connectPromise`, asserts
  `initResolved === false` after a microtask flush, then resolves the
  inner promise and asserts `initResolved === true`. This is the
  correct pattern for proving "we now wait" — the old test's
  `setTimeout(0)` could not have caught the race.

## Commentary

This is the right fix. The old fire-and-forget pattern caused a real
ordering bug: callers of `initializeApp` (notably the non-interactive
CLI entry points and several test harnesses) assumed that on resolve
the IDE state was usable, which was not true under the previous
implementation. Tests that ran fast enough would intermittently see
`isConnected() === false` even after `initializeApp` resolved.

Two concerns to raise before merge:

1. **Latency regression for IDE-mode startup.** Previously, the slow
   `connect()` round-trip happened in the background while the rest of
   the CLI continued to boot — the user got a prompt sooner, and the
   IDE features lit up a moment later. Awaiting it serially means
   first-prompt latency now includes the IDE handshake. For a healthy
   IDE this is sub-100 ms and fine; for a stuck/slow IDE socket this
   could feel like the CLI hangs at startup. A timeout
   (`Promise.race([connect, sleep(2000)])`) with a debug log on
   timeout would preserve responsiveness while still ensuring the
   connect attempt completes-or-fails before we hand control to the
   caller.

2. **Error swallowing semantics didn't change but are worth a second
   look.** Both old and new code only `debugLogger.error` on connect
   failure. With the new sequential await, a connect failure no longer
   causes `initializeApp` to reject — it just logs and returns the
   normal result. That matches the prior contract, but it does mean
   the new test's claim "we wait for IDE connection" is *also* "we
   silently swallow IDE connection failures and continue" — which is
   probably intentional but worth a comment in the source so future
   maintainers don't try to "fix" it by re-throwing.

3. **Test fixture nit.** The new test calls `await Promise.resolve()`
   to flush microtasks. That works for the current implementation but
   is fragile against a refactor that introduces a `setTimeout(0)` or
   an extra `await`. Consider `await new Promise(r => setImmediate(r))`
   or restructuring around `vi.useFakeTimers()` for stronger
   determinism.

Solid fix to a real race. Merge after the timeout and the inline
comment land.
