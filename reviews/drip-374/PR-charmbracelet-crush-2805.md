# charmbracelet/crush #2805 — fix(agent): drain queued messages after manual session summarize

- URL: https://github.com/charmbracelet/crush/pull/2805
- Head SHA: `1ebe35abf37b3f48a6e04791b85cee026b39d31b` (`1ebe35a`)
- Author: @taciturnaxolotl (Kieran Klukas)
- Diff: +18 / −1 (1 file: `internal/agent/agent.go`)
- Verdict: **merge-after-nits**

## What this changes (and why this is a real bug)

Fixes #1422. The `Summarize` method on `sessionAgent` previously
returned immediately after `a.sessions.Save(genCtx, currentSession)`
at line 734 without doing the queue-drain step that `Run()`'s
post-completion path does. Result: any messages the user typed while
a manual `/summarize` was in flight stayed pinned in
`a.messageQueue` until the *next* user message arrived to trigger
`Run()` again — at which point the queued messages would replay
ahead of the new message, with several seconds (or minutes) of
visible latency between user input and observable progress.

The fix at the new lines 735-754 does three things in order: (1)
returns early on `Save` error, (2) explicitly clears the active
request and cancels the summarize-scoped context (`a.activeRequests.Del(sessionID)
+ cancel()`), and (3) reads `messageQueue.Get(sessionID)`, pops the
head message, persists the tail back via `messageQueue.Set(sessionID, queuedMessages[1:])`,
and recursively calls `a.Run(ctx, firstQueuedMessage)`.

The active-request release ordering (step 2 before step 3) is the
load-bearing detail: `Run()` checks the busy state via
`activeRequests` and would refuse to start the queued message if the
release hadn't happened first. The author calls this out explicitly
in the new comment block at line 737: "Release the active request
before processing queued messages so that Run() does not see the
session as busy." Correct.

The recursive `Run(ctx, firstQueuedMessage)` is also the right shape
because `Run()` itself has the canonical post-turn drain loop — once
it finishes the first queued message, it'll naturally pick up the
remaining `[2:]` from the queue without `Summarize` needing to
re-enter. So this PR doesn't duplicate the drain logic; it just
re-enters the canonical path.

## Concerns

(a) **Context lifetime ambiguity.** `Summarize` accepts `ctx` from
its caller and presumably derives `genCtx` from it (the diff snippet
shows `a.sessions.Save(genCtx, currentSession)` but not where
`genCtx` is constructed); the `cancel()` call at line 745 is
canceling some scoped child context. The recursive `a.Run(ctx, firstQueuedMessage)`
at line 753 then uses the *original* outer `ctx`, which is correct
— but if the outer `ctx` is itself a request-scoped context that the
caller cancels as soon as `Summarize` returns, the recursive Run
will be canceled mid-turn. The PR's behavior here depends on
`/summarize`'s caller passing a context that outlives the return —
worth a regression test or an explicit assertion in the comment that
the recursive Run's ctx must outlive the original Summarize call.

(b) **No test added.** This is a `fix` PR for a real reported bug
(#1422) with non-trivial concurrency: queue drain, active-request
release, recursive Run dispatch. A table-driven test at
`internal/agent/agent_test.go` exercising "summarize with N≥1 queued
messages → assert Run was called with the right head message and
queue tail was correctly truncated" would be cheap to add and would
prevent a future refactor from silently re-introducing #1422. Right
now if someone reorders the active-request release vs. the queue pop
the test suite stays green.

(c) **Recursive Run error handling.** `qErr` from the recursive Run
is returned directly to the original `/summarize` caller as if it
were a Summarize error. From the user's perspective the failure mode
"summarize succeeded, but the queued-message replay errored" looks
identical to "summarize itself errored" — the user sees an error
toast on `/summarize` and won't realize the summary actually
completed and the failure was on a downstream message. Wrap the
recursive error: `return fmt.Errorf("queued message replay after
summarize failed: %w", qErr)` so logs distinguish.

(d) **`messageQueue.Set(..., queuedMessages[1:])` is non-atomic vs.
the `Get` at line 749.** If another goroutine appends to the queue
between the Get and the Set, the appended message gets clobbered.
The crush queue model's threading semantics aren't visible from this
diff — if `messageQueue` is meant to be single-writer per session
(plausible: only `Run` and `Summarize` mutate it for a given
sessionID, both gated by activeRequests), the `Get`/`Set` pair is
safe by construction. But a one-line `// safe by activeRequests
mutual exclusion` breadcrumb comment would save a future reader.

The fix is correct in shape and addresses the right bug. Ships once
the recursive-error wrap and a basic regression test land — both
mechanical.
