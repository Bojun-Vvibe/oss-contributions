# google-gemini/gemini-cli #26469 — Fixing race condition in task updates

- URL: https://github.com/google-gemini/gemini-cli/pull/26469
- Head SHA: `dc82d97b09ead3a71631896c428807a413ddb0ef`
- Author: kschaab
- Size: +181 / -16 across 4 files in `packages/a2a-server`

## Comments

1. `packages/a2a-server/src/agent/task-event-driven.test.ts` — Every `TOOL_CALLS_UPDATE` handler invocation now passes `schedulerId: 'task-id'`. Good — this matches the new producer contract. But `'task-id'` is hard-coded in 12+ places; consider a small `makeUpdate(toolCalls)` helper so a future schedulerId rename doesn't churn the test file again.
2. `packages/a2a-server/src/agent/task.test.ts` (new `Race Condition Fix` block, ~line 463+) — Excellent: the test directly exercises `_registerToolCall(..., 'validating')` and asserts `taskState !== 'input-required'`. Verify the assertion isn't satisfied by an unrelated state (`assert.equal(task.taskState, 'submitted')` would be tighter than `not.toBe('input-required')`).
3. `packages/a2a-server/src/agent/task.ts` — Diff truncated; please confirm the producer side now actually sets `schedulerId` on every `TOOL_CALLS_UPDATE` it publishes, otherwise downstream subscribers in other packages will see `undefined` and silently misroute.
4. The PR title says "race condition in task updates" but the test only covers the validating-vs-awaiting-approval transition. If the schedulerId addition closes a *separate* cross-task contamination bug, please add a test that publishes a `TOOL_CALLS_UPDATE` with a non-matching schedulerId and asserts the handler ignores it.
5. `packages/a2a-server/src/http/app.test.ts` — Diff truncated; ensure HTTP-level test still passes both before and after by smoke-testing one happy-path flow.
6. Backwards compatibility: are there any in-flight `TOOL_CALLS_UPDATE` producers (e.g. plugins) that won't yet send `schedulerId`? If so, the consumer should treat missing `schedulerId` as a wildcard-match for one release before becoming strict.

## Verdict

`merge-after-nits`

## Reasoning

The fix targets a real concurrency hole: a tool call mid-validation could trigger an `input-required` transition because the previous logic counted only `awaiting_approval`. The new test wires real internal state (`_registerToolCall`) to assert the guard, which is the right level. Two follow-ups would harden this: (a) the schedulerId routing path needs an explicit "ignore other tasks' updates" test, (b) tighten the negative assertion to a positive expected state. Otherwise this looks like a correctness-positive change.
