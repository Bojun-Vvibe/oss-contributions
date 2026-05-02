# Review — openai/codex#20790

- PR: https://github.com/openai/codex/pull/20790
- Title: [codex] Keep paused goals paused on thread resume
- Head SHA: `d08d3ba0892c80e6ca52f40caf58f1e550a3b39a`
- Size: +28 / −78 across 3 files
- Verdict: **merge-as-is**

## Summary

Behavioural fix in `codex-rs/core/src/goals.rs`: previously, when a
thread was resumed (`GoalRuntimeEvent::ThreadResumed`), the runtime
would auto-flip a `Paused` goal to `Active` and emit a
`ThreadGoalUpdated` event, immediately scheduling another
continuation turn. That made it impossible to pause a goal and have
the pause survive a session reload — the next resume would silently
restart autonomous work. This PR renames the entry point to
`restore_thread_goal_runtime_after_resume` and makes it pure-
restorative: it only re-marks accounting state for goals that were
already `Active`, and clears wall-clock state for `Paused`,
`BudgetLimited`, or `Complete` goals. No status mutation, no
synthetic goal-updated event, no continuation scheduled.

## Evidence

- `codex-rs/core/src/goals.rs:343` — the `ThreadResumed` arm now
  calls `restore_thread_goal_runtime_after_resume` (return type
  `()`), down from the old `activate_paused_thread_goal_after_resume`
  which returned `bool` and could trigger a state mutation.
- `codex-rs/core/src/goals.rs:1020-1060` — the new body is a clean
  `match goal.status { Active => mark_active_goal, Paused |
  BudgetLimited | Complete => clear_stopped_thread_goal_runtime_state }`,
  versus ~70 lines of conditional `update_thread_goal` + event
  emission + `mark_active_goal_accounting` that the old code ran.
- Test rename `thread_resume_emits_active_goal_update_before_continuation`
  → `thread_resume_keeps_paused_goal_paused`
  (`codex-rs/app-server/tests/suite/v2/thread_resume.rs:388`) flips
  the assertion: now expects `ThreadGoalStatus::Paused` after resume
  and asserts no `turn/started` notification fires.
- `codex-rs/core/src/thread_manager_tests.rs:1126` —
  `resumed_thread_keeps_paused_goal_paused` similarly asserts
  `active_turn` remains `is_none()` after resume of a paused-goal
  thread.
- The doc-comment on `goal_runtime_apply` (lines 270-279) is updated
  to say "thread resumes restore runtime state for already-active
  goals" instead of "resumes reactivate paused goals" — the spec
  matches the new behaviour.

## Notes

This is an intentional UX change with explicit test coverage that
inverts the prior behaviour. The new behaviour is unambiguously
better: pause means pause, including across resumes; if a user wants
to continue a paused goal after resume, they go through the explicit
maybe-continue path. The diff removes more than it adds, and removes
the trickiest piece (the synthetic `ThreadGoalUpdated` emission that
ran during resume snapshot replay). Ship.
