# openai/codex #20657 — Resume queued prompts after blocked UserPromptSubmit hook

- URL: https://github.com/openai/codex/pull/20657
- Head SHA: `39cc5843d36ee0fb3ec6b0746e4d323c47eac863`
- Files: `codex-rs/tui/src/chatwidget.rs` (+11/-0), `codex-rs/tui/src/chatwidget/tests/app_server.rs` (+78/-0)

## Context / problem

The TUI prompt queue is drained one-at-a-time by `maybe_send_next_queued_input` (`chatwidget.rs:6696-6740` per PR body). It refuses to dequeue another item while `user_turn_pending_start` is `true`. Normally the next-turn lifecycle event clears that latch — but a `UserPromptSubmit` hook that returns `Blocked` or `Stopped` short-circuits the prompt before any turn actually starts. The pre-existing `on_hook_completed` (`chatwidget.rs:4016-4043`) only flushed hook output and never released `user_turn_pending_start` in the blocked/stopped arm. Result: the *first* blocked queued prompt strands all subsequent queued prompts. User-visible symptom: after one hook-blocked prompt, the queue silently freezes; later prompts never submit even though the active turn never began.

## Design analysis

Fix at `chatwidget.rs:4014-4017`:
```rust
let blocked_user_prompt_submit = self.user_turn_pending_start
    && completed.event_name == codex_app_server_protocol::HookEventName::UserPromptSubmit
    && matches!(
        completed.status,
        codex_app_server_protocol::HookRunStatus::Blocked
            | codex_app_server_protocol::HookRunStatus::Stopped
    );
```
captured *up front* (before the body that mutates `active_hook_cell`), then at `:4046-4049` after the existing render path completes:
```rust
if blocked_user_prompt_submit {
    self.user_turn_pending_start = false;
    self.maybe_send_next_queued_input();
}
```

Three things this gets right:

1. **Captured-before, applied-after** — the `blocked_user_prompt_submit` decision uses `self.user_turn_pending_start` as observed at function entry, not after the existing body has potentially mutated state. So if some other path inside `on_hook_completed` already cleared the latch, we don't double-process.
2. **Conjunctive guard pinned to the exact failure mode** — the predicate requires (a) `user_turn_pending_start` set, (b) the hook event was specifically `UserPromptSubmit` (not some other hook event whose blocked completion is irrelevant), (c) status is `Blocked` *or* `Stopped`. Both terminal non-success states for a hook that prevents the turn from starting. `Failed` is intentionally not included — that's a different recovery path (the turn may still proceed or surface an error).
3. **Re-pumping the queue immediately** — calling `maybe_send_next_queued_input()` in the same tick rather than relying on the next render cycle means the next queued prompt fires before the user notices anything stuck.

The regression test at `app_server.rs:170-248` exercises exactly the failure mode the PR description names: queue two prompts ("blocked prompt", "allowed prompt") via `Tab` after a `TurnStarted`/`TurnCompleted` cycle, then deliver a `Blocked` `UserPromptSubmit` hook completion for the first, and assert the second is submitted. Naming the test `blocked_user_prompt_submit_hook_continues_queued_inputs` is good — when this regresses in the future, the test name will describe the symptom precisely.

## Risks

- **`Stopped` semantics** — the PR body conflates `Blocked` and `Stopped` into "the turn won't start." Worth a one-line PR comment confirming `HookRunStatus::Stopped` cannot leave any partial turn state behind that the next queued prompt would step on (e.g. partial `running_command_started_at` entries from the prior tick — see #20653's HashMap discipline). If `Stopped` can occur *mid-turn-startup*, this fix may release the latch while some other state is still live.
- **No test for the negative arm** — the test covers "blocked → next prompt submitted." There's no test for "hook `Failed` → latch *not* cleared (the in-flight turn will still own it)" or "non-`UserPromptSubmit` hook blocked → latch *not* cleared." Both are easy to add and lock the conjunctive predicate so a future contributor doesn't widen the guard accidentally.
- **`active_hook_cell` ordering** — the new clear runs *after* `flush_completed_hook_output()` and `finish_active_hook_cell_if_idle()`, which is correct (UI cell finishes its lifecycle first). But `request_redraw()` is the very last call; the queue dispatch happens before redraw. If `maybe_send_next_queued_input()` itself triggers any state that needs a redraw, it should be fine because the trailing `request_redraw()` covers it — worth an inline comment so the next maintainer doesn't reorder these.

## Suggestions

- Inline comment at `:4046` explaining *why* this clear happens here rather than in `maybe_send_next_queued_input` itself: "the hook-blocked-pending-turn case is the one path where no `TurnStarted` will ever fire, so we have to clear the latch here on the hook-completion edge."
- Add a sibling test for the non-blocked completion of a `UserPromptSubmit` hook (status `Allowed` or whatever the success enum is) asserting `user_turn_pending_start` is *not* cleared by this path — proves the conjunctive guard isn't accidentally widened.
- Consider extracting `HookRunStatus::Blocked | HookRunStatus::Stopped` into a helper like `HookRunStatus::is_terminal_non_success()` or `prevents_turn_start()` — naming the predicate at the type level reads better and keeps the truth table in one place.

## Verdict

`merge-after-nits` — surgical 11-line fix for a real "queue silently stalls after one hook-blocked prompt" symptom, captured at the right boundary (`on_hook_completed`'s edge), with a captured-before-applied-after pattern that doesn't fight the existing body, a conjunctive guard pinned to the exact failure shape, immediate re-pump rather than next-tick, and a dispositive regression test. Wants the negative-arm test, the predicate-extraction nit, and a one-liner on `Stopped` semantics.
