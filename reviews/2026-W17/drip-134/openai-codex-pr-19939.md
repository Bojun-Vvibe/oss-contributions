# openai/codex #19939 — Restore TUI working status after pending steer

- PR: https://github.com/openai/codex/pull/19939
- Head SHA: `f2d626134bc97a2a2b24137fe366ddd604fb0587`
- Files changed: 2 (`+59/-1`) — `codex-rs/tui/src/chatwidget.rs` (+1/-1), `codex-rs/tui/src/chatwidget/tests/status_and_layout.rs` (+58)
- Fixes: #19925

## Verdict: merge-as-is

## Rationale

- **One-line conditional flip with the correct condition.** `chatwidget.rs:4949-4952` changes the `MessagePhase::FinalAnswer | None` arm of `pending_status_indicator_restore` from unconditional `false` to `!self.pending_steers.is_empty()` — exactly right. The previous behavior assumed final-answer completion always meant the turn was idle and the working spinner should stay hidden; with a steer queued during streaming, the turn is *not* idle and the spinner needs to come back so the user knows the steer is being processed. The `Commentary` arm stays at unconditional `true` (correct — commentary completion always implies more output is coming so the spinner restores).
- **Test pins the load-bearing 4-state cell exactly.** `status_and_layout.rs:960-1016` runs the full reproduction sequence: `on_task_started` → spinner visible, two `on_agent_message_delta` + `on_commit_tick` cycles → spinner hidden but `is_task_running()==true` (assertion at `:982-983`), submit a steer via `set_composer_text` + Enter → `pending_steers.len() == 1` (`:992`), `complete_assistant_message(..., MessagePhase::FinalAnswer)` → spinner *visible* + task running (`:1010-1011`, the new invariant), then `complete_user_message` for the steer → `pending_steers.is_empty()` and spinner stays visible (`:1014-1016`, pinning that the steer-completion path doesn't re-flip the indicator). This is the right shape — testing both the transition (the bug) and the post-transition steady state (so a future "let me simplify by always returning false here" change breaks loudly).
- **`pending_steers` is the correct signal.** It's the field that tracks "user submitted input mid-turn that hasn't been turned into its own server-side turn yet," which is exactly the lifecycle that needs to keep the spinner visible after the in-flight final answer lands. Using `!is_empty()` rather than e.g. checking `is_task_running()` is correct because `is_task_running()` is broader (covers any pending response) and would over-restore the spinner in the no-steer case.
- **Test reuses existing harness primitives** (`make_chatwidget_manual`, `drain_insert_history`, `next_submit_op`, `complete_assistant_message`, `complete_user_message`) so it doesn't introduce new mocking surface — and `next_submit_op(&mut op_rx)` matches `Op::UserTurn { items, .. }` and asserts the expected `UserInput::Text` content (`:984-993`) which adds bonus coverage that the steer-submission op-channel side is unchanged by this fix.

## Nits / follow-ups

- Comment at `:4951` (`// Models that don't support preambles only output AgentMessageItems on turn completion.`) is now slightly stale relative to the new condition — worth a one-line addendum like "Restore the spinner if a steer is queued, since the next turn is about to start." A future reader scanning the comment sees only the old motivation.
- No corresponding test for the `Commentary` arm changing simultaneously — but that arm wasn't touched, so this is fine.
