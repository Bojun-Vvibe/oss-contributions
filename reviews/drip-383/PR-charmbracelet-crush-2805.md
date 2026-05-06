# charmbracelet/crush#2805 — fix(agent): drain queued messages after manual session summarize

- **Head SHA**: `1ebe35abf37b3f48a6e04791b85cee026b39d31b`
- **Author**: taciturnaxolotl (Kieran Klukas)
- **Stats**: +18 / -1, 1 file (fixes #1422)

## Summary

When `Summarize()` is invoked manually (not via Run's compaction trigger), the session early-exits the message loop and any messages enqueued during the summarize never get flushed — they stay queued until the *next* user-initiated message kicks Run() awake. PR drains the queue at the tail of `Summarize()` by releasing the active-request handle and calling `Run()` with the first queued message.

## Specific citations

- `internal/agent/agent.go:735-752` (in `1ebe35a`): the rewrite turns the bare `return err` into a multi-step:
  1. `:735-738` — bail if save returned an error (preserves prior behavior).
  2. `:742-743` — `a.activeRequests.Del(sessionID); cancel()` — releases the busy lock and cancels the summarize-scoped context *before* re-entering `Run()`. Comment explicitly names the rationale ("so that Run() does not see the session as busy").
  3. `:746-748` — early-return if no queued messages.
  4. `:749-752` — pop `queuedMessages[0]`, write back `queuedMessages[1:]` to the queue, call `a.Run(ctx, firstQueuedMessage)`, return its error.

## Verdict

**merge-after-nits**

## Rationale

The bug is real and the fix is structurally correct — releasing `activeRequests` *before* the recursive `Run()` is essential, otherwise `Run()` would see the session still busy and double-queue. Concerns: (1) Only the *first* queued message is drained per Summarize call. If multiple messages were queued (likely, since summarize is triggered when messages pile up), the recursive `Run()` must itself drain the rest; this isn't explicit in the diff. The author may be relying on `Run()`'s existing tail-drain logic but a test would prove it. (2) The `cancel()` call at `:743` cancels `genCtx` (the summarize-scoped context) but the recursive `a.Run(ctx, ...)` uses the *outer* `ctx` — correct, but worth a comment naming which context is being cancelled and why. (3) No test added; a unit test queuing 2 messages → triggering summarize → asserting both drain would harden this. (4) The non-zero-return-without-cleanup path: if the recursive `Run()` returns an error (`qErr`), the queue has already been mutated to `[1:]` so the popped message is lost — acceptable for "manual summarize" UX but worth a log line.

