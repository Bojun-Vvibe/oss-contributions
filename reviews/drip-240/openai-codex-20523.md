# openai/codex#20523 — Remove no-tool goal continuation suppression

- **PR**: https://github.com/openai/codex/pull/20523
- **Head SHA**: `f5a1455714169f17be3ec0b014d2e488da807bfe`
- **Verdict**: `merge-after-nits`

## What it does

Deletes a continuation-suppression heuristic that stopped `/goal` autonomy
after any continuation turn that produced zero registry tool calls. Three
removals together: (1) the prompt sentence telling the model to wait for
new input when blocked, (2) the runtime `continuation_suppressed:
AtomicBool`, (3) the seven `reset_thread_goal_continuation_suppression()`
call sites that maintained it. +141 / -64.

## What's load-bearing

- `codex-rs/core/src/goals.rs:32-33` removes
  `use std::sync::atomic::AtomicBool` + `Ordering`, structurally signaling
  the entire suppression primitive is gone (no other field in
  `GoalRuntimeState` uses atomics).
- `codex-rs/core/src/goals.rs:91` deletes `tool_calls: u64` from the
  `TurnFinished` event variant. This is the *plumbing* the suppression
  decision needed; deleting it at the event boundary means no future
  caller can accidentally re-introduce the heuristic by reading a count
  that no longer exists. Right discipline.
- `codex-rs/core/src/goals.rs:728-744` collapses
  `finish_thread_goal_turn` from the prior "if continuation turn AND zero
  tool calls THEN suppress" branch to a flat
  `take_thread_goal_continuation_turn(&turn_context.sub_id).await;` —
  three lines, no decision.
- `codex-rs/core/src/goals.rs:1217-1227` removes the suppression check in
  the continuation scheduler that previously logged "skipping active goal
  continuation because the last continuation made no tool calls" and
  returned `None`. This is the actual user-visible bug fix: the scheduler
  no longer short-circuits.
- `codex-rs/core/templates/goals/continuation.md:1` — single-character
  template change (the "explain the blocker or next required input to the
  user and wait for new input" sentence is removed). Pinned by the
  template-rendering test at `goals.rs:1530-1534` flipping
  `assert!(prompt.contains(...))` to `assert!(!prompt.contains(...))`.
- `codex-rs/core/src/session/tests.rs:6944-7050` adds two new regressions:
  `active_goal_continuation_runs_again_after_no_tool_turn` exercises the
  exact bug (assistant message with no tools → continuation must still
  fire), and `pending_request_user_input_does_not_spawn_extra_goal_continuation`
  pins the "but `request_user_input` keeps the existing turn open instead
  of spawning a continuation" guarantee. Both tests target the right
  semantic boundary (waiting for `TurnComplete` events with a count rather
  than reading internal state).

## Concerns

1. **The deleted heuristic existed for a reason; PR body acknowledges it
   ("brittle heuristic that treated 'no registry tool calls' as
   equivalent to 'should stop'") but doesn't say what protects against
   the failure mode it was guarding against.** Without the suppression,
   a model that emits a chat-only "I am still working" assistant message
   on every continuation turn will continuation-loop indefinitely against
   the budget cap. Is the budget the only safety net now? The PR should
   call out which other guard catches the "model does nothing useful but
   keeps generating prose" case, or document that this is intentional and
   the budget is the sole stop condition.
2. **Test #1 (`runs_again_after_no_tool_turn`) only asserts 3 turns
   completed, not that turns *terminate*.** A bounded-iteration test
   with an assertion that the goal *eventually completes* (resp-4 fires
   `update_goal status=complete`) would be a stronger pin against the
   "loops forever" regression. The current shape would pass even if the
   continuation kept firing past the asserted count.
3. **Removed prompt sentence ("explain the blocker… wait for new input")
   was the only user-facing instruction for the genuine "I'm stuck" case
   too.** Without it, a model that *is* legitimately blocked has no
   prompt-level guidance to surface that to the user. Consider replacing
   with a positive instruction like "If you cannot make further progress
   without user input, call `request_user_input` with a specific
   question" — the PR body suggests `request_user_input` is the intended
   path but the prompt no longer points at it.

## Verdict

`merge-after-nits` — the structural cleanup (deleting `AtomicBool` +
event-payload field at the source) is the right discipline, and both new
tests target real semantic boundaries. The "what stops the
chat-only-loop case" question (#1) is the load-bearing one and deserves a
PR-body answer before merge.
