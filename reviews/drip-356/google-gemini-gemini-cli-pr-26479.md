# google-gemini/gemini-cli#26479 — fix(a2a-server): resolve tool approval race condition and improve status reporting

- Head SHA: `a7f309adb46349df98b97eafcf1e54102a710072`
- Author: kschaab
- Verdict: **merge-after-nits**

## Summary

Fixes a race in the A2A server task lifecycle where a task could
transition to `input-required` while tool calls were still being
validated or scheduled. Threads a `schedulerId` through the
`TOOL_CALLS_UPDATE` message-bus event so a multi-scheduler task can
correctly partition pending tool calls per scheduler. Adds tighter
state-tracking in `Task.checkInputRequiredState` to count
`validating` and `scheduled` tool calls (not just `awaiting_approval`)
plus tools the policy engine has already auto-confirmed.

## Specific references

- `packages/a2a-server/src/agent/task.ts` — `checkInputRequiredState`
  expanded to include `validating` and `scheduled` states; previously
  only `awaiting_approval` was counted, which is why the task could
  flip to `input-required` while a tool was mid-validation.
- `packages/a2a-server/src/agent/task-event-driven.test.ts:67-70`,
  `:108-111`, `:152-156`, `:182-186`, `:227-231`, `:269-273`,
  `:312-316` — every existing `TOOL_CALLS_UPDATE` test fixture grows
  a `schedulerId: 'task-id'` field. The breadth of this change shows
  `schedulerId` is now load-bearing on the message contract; old
  publishers that don't include it will be classified as "not mine"
  by every subscriber.
- `packages/a2a-server/src/http/app.test.ts` — HTTP-layer tests
  updated alongside, suggesting full-stack coverage for the new
  state-tracking.

## Reasoning

The race description is plausible and the fix shape is right:
state-machine code that counts "tools currently in flight" must
count *all* in-flight states, not just one. Adding `schedulerId` to
the event payload also lets a task with multiple concurrent
schedulers (e.g., a parent task spawning sub-task schedulers) avoid
crosstalk where one scheduler's TOOL_CALLS_UPDATE would mis-trigger
state recompute on another.

Nits to address before merge:

1. **Backward-compatibility of `schedulerId`.** Every test in this
   diff sets `schedulerId: 'task-id'`. What does the production code
   do when an old event arrives without it? If `schedulerId` is now
   required at the type level but absent at runtime, the subscriber
   will throw or silently drop the event. Either make it required
   at the publisher (and confirm all in-tree publishers were
   updated — the diff is mostly tests, so the publisher coverage is
   hard to verify from the diff alone), or keep it optional with a
   clear "if undefined, treat as broadcast" semantic.

2. **Naming.** `schedulerId: 'task-id'` in the tests reads as if a
   task ID is being plugged into a scheduler-ID slot. If that's
   intentional (one scheduler per task), document it; if it's a
   test shortcut, give the test a real distinct value so future
   readers don't infer the wrong invariant.

3. **Test coverage of the actual race.** The test diff updates
   existing payload shape but I don't see a new test that
   reproduces the original race (tool in `validating`, status
   incorrectly flips to `input-required`). Add at least one
   regression test that asserts the task stays in
   `working`/`tool_running` for the full window during which any
   tool is in `validating` or `scheduled`.

4. **`improve status reporting`** half of the title is
   under-described in the PR body — flag what user-visible status
   values changed so client teams can update any string-matching
   they have.

The core fix is well-targeted; merge after the back-compat semantic
on `schedulerId` is pinned down.
