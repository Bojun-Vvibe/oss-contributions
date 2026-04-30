# QwenLM/qwen-code#3777 — fix(test): restore abort-and-lifecycle stdin-close test to pre-#3723 version

- PR: https://github.com/QwenLM/qwen-code/pull/3777
- Head SHA: `b5f68091360b36685dcec9c61db21d77f0780fca`
- Author: wenshao (Shaojin Wen)
- Files: `integration-tests/sdk-typescript/abort-and-lifecycle.test.ts`
  +23/−12

## Context

The E2E test `should handle control responses when stdin closes
before replies` started timing out 900s (5min × 3 retries) on
both macOS and Linux runners after #3723 merged. The PR author
(wenshao) traced this to one specific commit inside #3723 that
rewrote the test in a way that flipped its semantics from
"verify the CLI handles control responses correctly when stdin
closes mid-flight" to "verify the LLM emits write_file → reply
'done' deterministically." The latter requires CI's LLM endpoint
to be deterministic about a 4-step tool-call sequence; it isn't.

## Design

The diff at `abort-and-lifecycle.test.ts:317-422` restores the
pre-#3723 test shape exactly:

1. **Reintroduces the `inputStreamDonePromise` latch** at
   `:328-335`, with a 15s reject timeout (defensive, gives a
   real error message instead of test-runner timeout). The
   latch was the load-bearing primitive — `canUseTool`
   resolves it, then the input generator awaits it before
   closing stdin, which is what creates the "control response
   in flight when stdin closes" race the test is named for.
2. **Restores `await inputStreamDonePromise`** in the async
   generator at `:370` after the second yield. Pre-#3723:
   stdin stays open until the canUseTool callback fires, then
   immediately closes. Post-#3723: stdin stayed open until
   `q.endInput()` after `secondResultPromise`, which required
   the full tool-call dance to complete first.
3. **Restores the `setTimeout(resolve, 1000)`** inside
   `canUseTool` at `:381-383`. The 1-second sleep is what
   keeps the control reply *in flight* when the input
   generator races ahead and closes stdin. Without it, the
   reply lands before the stdin close and the race the test
   exists to catch never happens.
4. **Restores the inverted final assertion** at `:419-420`:
   `expect(content).toBe('original content')`. Pre-#3723:
   the test's invariant was "if stdin closes mid-control-
   reply, the in-flight tool call is *not* executed."
   Post-#3723: "the tool call *is* executed" — which is the
   opposite invariant and depends on the LLM completing the
   full sequence.
5. **Replaces the LLM-tool-call prompt** at `:365-366` with a
   minimal `'Write "updated" to test.txt. Stop if any
   exception occurs.'` — the test no longer needs the LLM to
   actually do the write because the assertion now is "the
   write *didn't* happen".
6. **Removes `q.endInput()`** at the old `:404` — stdin closes
   naturally via the generator returning, which is the
   pre-#3723 shape and the actually-tested code path.

## Evidence

PR description cites two CI runs (`4cd9f0cb` = #3723 itself,
`8b6b0d64f` = #3771) both failing 900s on the same test on
both platforms, plus 30 prior runs stably green. That's
sufficient to attribute the flake to the #3723 test rewrite
specifically and not to a #3723 core-code regression — the
core changes (`permissionFlow.ts`, `coreToolScheduler.ts /
Session.ts` rewiring, the `npm run bundle` step in the E2E
workflow) are all kept intact.

## Risks / nits

- The 1-second `setTimeout` inside `canUseTool` is the kind
  of thing that gets "tidied" by a future refactor that
  doesn't grok the race. A 2-line comment at `:381` —
  `// Hold the canUseTool reply in flight long enough for
  the input generator to close stdin; the race is the
  point of this test` — would be cheap insurance. Worth
  asking the author to add.
- The `setTimeout(reject, 15000)` on the new
  `inputStreamDonePromise` at `:332-334` mirrors the same
  shape on `canUseToolCalledPromise` at `:325-327`. Good
  symmetry, but if the test's stdin-close→reply ordering
  actually takes >15s in slow runner conditions, the
  rejection will mask the original failure. 15s is
  probably fine given the 1s sleep inside; worth checking
  one full E2E run on the slowest platform (macOS) before
  committing.
- The test currently calls `console.log(JSON.stringify(_message,
  null, 2))` at `:381` — kept from the post-#3723 version.
  That's fine for E2E debug logging but produces noisy
  output; if the maintainer prefers clean test output, a
  conditional `if (process.env.DEBUG_E2E)` wrapper would
  let it stay without polluting the default run.
- The PR explicitly scopes out the macOS-only flake of
  `interactive/cron-interactive.test.ts > loop fires inline
  in conversation` — correct decision, separate root cause.

## Verdict

**`merge-as-is`**

This is the right shape: revert *only* the broken test file
to its pre-#3723 form, keep all core changes intact, ship
defensive reject-timeouts on the two awaitable promises so
the next failure produces a real error message instead of a
test-runner 900s timeout. Evidence is concrete (named CI
runs, before/after pattern). Net unblocks the E2E workflow
which is currently red on every push to main.

## What I learned

When a PR's "fix the test" commit flips the test's
semantic meaning (here: from CLI-robustness to LLM-
determinism), the test stops measuring what its name
claims. The clearest tell is when the new assertion shape
requires upstream non-deterministic behavior that the
pre-PR assertion didn't. Restoring the test to its
original semantics is the right fix; trying to make the
LLM deterministic would be the wrong one.
