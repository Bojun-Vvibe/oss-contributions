# Review: QwenLM/qwen-code#3777 — fix(test): restore abort-and-lifecycle stdin-close test to pre-#3723 version

- PR: https://github.com/QwenLM/qwen-code/pull/3777
- Head SHA: `b5f68091360b36685dcec9c61db21d77f0780fca`
- State: OPEN
- Files: `integration-tests/sdk-typescript/abort-and-lifecycle.test.ts` (+23/-12)
- Verdict: **needs-discussion**

## What it does

Rewrites the integration test `should handle control responses when stdin
closes before replies` in two ways that change *what is being asserted*,
not just how:

1. Adds 15-second timeouts to the `canUseToolCalledPromise` and a new
   `inputStreamDonePromise`, so a hung await doesn't hang the entire CI
   job.
2. Changes the test's *expectation* from `expect(content).toBe('updated')`
   (the file got written by the agent) to `expect(content).toBe('original
   content')` (the file did *not* get written), and removes the
   `expect(canUseToolCalled).toBe(true)` assertion. Also removes the
   explicit `q.endInput()` call between `secondResultPromise` and the
   file-content check.

The PR title frames this as "restore to pre-#3723 version" — i.e., #3723
(the "shared permission flow for tool execution unification" feature)
changed the test's expected behaviour, and this PR is rolling that
expectation back.

## Notable changes

- `abort-and-lifecycle.test.ts:316-330` — adds `inputStreamDonePromise`
  with a 15-second `setTimeout(reject, 15000)` and a matching timeout on
  `canUseToolCalledPromise`. The promise is rejected with a descriptive
  `Error('canUseTool callback not called')` / `Error('inputStreamDonePromise
  timeout')`. This is the right shape for a CI-friendly hang protection —
  the rejection surfaces as a test failure with a usable message rather
  than a job timeout 30 minutes later.
- `:362-371` — the prompt sent to the model changes from `Use the
  write_file tool to write "updated" to the file at ${testFilePath}. Then
  reply with "done"` to `Write "updated" to test.txt. Stop if any
  exception occurs.` Two semantic shifts here: (a) drops the absolute
  path in favour of bare filename, and (b) appends "Stop if any exception
  occurs," which is a model-directed instruction to halt on error. Both
  changes increase the variance of model behaviour on this test.
- `:367` — adds `await inputStreamDonePromise;` immediately after enqueueing
  the user message. This is the load-bearing change for the new
  expectation: by awaiting the `inputStreamDoneResolve()` call (which is
  fired from inside `canUseTool`), the input-stream generator now blocks
  *before* the model can complete the tool execution flow. The 1-second
  `setTimeout` inside `canUseTool` at `:382` then gives time for the
  stdin-close to propagate before `canUseToolCalledResolve()` allows the
  test to proceed to the assertion.
- `:382-385` — `canUseTool` callback is rewritten to call
  `inputStreamDoneResolve()` first, sleep 1s, then resolve
  `canUseToolCalledPromise`. The `await new Promise(resolve =>
  setTimeout(resolve, 1000))` is the synchronization seam — it deliberately
  keeps the canUseTool async stack alive past the stdin-close window.
- `:418-421` — removes `q.endInput()` and replaces `expect(content).toBe(
  'updated')` with `expect(content).toBe('original content')`. Removes the
  `canUseToolCalled` boolean tracking entirely.

## Reasoning

This is where the discussion is needed. The PR is asserting that after
#3723 (shared permission flow for tool execution unification), the
intended behaviour of the "stdin closes before replies" test is that
**the file is not written at all** — the tool call is initiated, the
permission callback fires, but the actual `write_file` execution is
abandoned because of the stdin-close coordinated with the awaited
`inputStreamDonePromise`.

Before #3723, the test expected the file *was* written (`'updated'`) and
that `canUseTool` was called. After this PR, the test expects neither —
the model recognized an abort condition (the "Stop if any exception
occurs" prompt nudge plus the stdin-close signal) and stopped before
mutating disk.

The framing as "restore to pre-#3723 version" is misleading: the
*expectations* are different from both pre-#3723 and post-#3723 baselines.
Pre-#3723 likely expected `'updated'` (test harness wrote the file via
the permitted tool call); post-#3723 broke that assertion in some way the
PR doesn't describe. This PR's expectation (`'original content'`) is a
*new* contract — "the agent should not have written the file when stdin
closes mid-tool-execution" — which is a defensible permission/abort
contract but is being introduced under a "test restoration" framing.

The 1-second `setTimeout` at `:384` is a wall-clock race condition. If the
CLI's stdin-close handling is faster than 1 second on a slow CI runner the
test will flake (file gets written before the abort propagates); if it's
slower the test will pass spuriously. Wall-clock sleeps in tests asserting
abort semantics are a well-known anti-pattern; this should be a
deterministic signal (e.g., a hook that fires when the input stream
actually closes).

The `console.log(JSON.stringify(_message, null, 2))` at `:399` looks like
left-in debug code. The variable rename to `_message` (leading underscore
convention for "intentionally unused") is then contradicted by the
`console.log` actually using it, which is confusing.

## Suggestions before merge

1. **Justify the new expectation in the PR body or commit message.** What
   is the contract being asserted? Is it "file is never mutated when
   stdin closes during permission resolution"? Make that explicit so a
   future reader doesn't try to "fix" this test back.
2. **Replace the 1-second sleep at `:384` with a deterministic signal.**
   Wait on a "stdin closed" event from the CLI rather than a timer.
3. **Remove the `console.log` at `:399` or gate it behind a debug flag.**
4. **Reconsider the prompt change at `:365`.** "Stop if any exception
   occurs" is testing model interpretation of natural language abort
   semantics, not the CLI's lifecycle handling. The original prompt was
   more deterministic.
5. **Add a comment at the top of the test explaining the permission flow
   sequence**: stdin-close → input-stream-done → 1s settle → canUseTool
   resolves → assertion. The current sequence requires reading three
   promise definitions in three different scopes to reconstruct.

If the new expectation is in fact the post-#3723 intended contract, this
test is fine after the nits above. If it's a workaround for a flake whose
root cause is in the CLI's permission flow, the CLI fix should land first
and this test should pin the *correct* behaviour, not accommodate the
broken one.
