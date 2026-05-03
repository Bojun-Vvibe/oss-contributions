# openai/codex PR #20838 — Pause goals while in Plan mode

- **Repo:** openai/codex
- **PR:** #20838
- **Head SHA:** `94d645330d2119035e528a70684a1f1ab8ff3242`
- **Author:** etraut-openai
- **Title:** Pause goals while in Plan mode
- **Diff size:** +430 / -357 across ~20 files (Rust)
- **Drip:** drip-295

## Files changed (substantive subset)

- `codex-rs/core/src/goals.rs` (+5/-134) — removes the entire `should_ignore_goal_for_mode(...)` family of guards and the `activate_paused_thread_goal_after_resume()` helper. Also drops the `GoalRuntimeEvent::ThreadResumed` arm of `goal_runtime_apply`.
- `codex-rs/core/src/codex_thread.rs` (-7), `codex-rs/core/src/thread_manager.rs` (-6), `codex-rs/app-server/src/codex_message_processor.rs` (-6) — wiring removals for the now-deleted `ThreadResumed` event.
- `codex-rs/tui/src/app/thread_goal_actions.rs` (+36/-9) — every goal mutation site now reads `chat_widget.is_plan_mode_active()` and pipes the requested status through `goal_status_for_plan_mode(status, plan_mode_active)` before calling `app_server.thread_goal_set(...)`. Adds a `show_thread_goal_updated(&goal)` helper that uses the new plan-mode hint renderer.
- `codex-rs/tui/src/chatwidget.rs` (+19/-6) — exposes `is_plan_mode_active()`; passes the hint flag into `goal_summary_lines(...)`.
- `codex-rs/tui/src/chatwidget/goal_menu.rs` (+19/-19) — UI label changes for the pause-in-plan-mode state.
- `codex-rs/tui/src/bottom_pane/footer.rs` (+20/-2) — adds `goal_paused_in_plan_mode_line(usage)` rendered in the status footer.
- `codex-rs/app-server/tests/suite/v2/thread_resume.rs` (+158/-154) — new `thread_resume_emits_paused_goal_snapshot_without_continuation` test (line 78) and a Plan-mode resume case (line 252).
- New TUI test files: `chatwidget/tests/goal_menu.rs` (+36), `chatwidget/tests/plan_mode.rs` (+48), `chatwidget/tests/status_and_layout.rs` (+46), plus three new insta snapshots.

## Specific observations

- `goals.rs:507-516` — the doc comment is updated to drop "plan mode ignores continuations" and "resumes reactivate paused goals". Good. But the bullet list now reads "interrupts pause active goals, maybe-continue events start idle goal continuation turns" — there is no mention of how plan-mode entry/exit interacts with the runtime. After this PR the policy is enforced at the *TUI call-site* (`thread_goal_actions.rs:75-79`), not in `Session::goal_runtime_apply`. That asymmetry deserves an explicit note in this comment, otherwise the next reader of `goals.rs` will assume the runtime still owns mode-awareness and write a non-TUI client (eg. the upcoming app-server runtime extension in #20814) that bypasses the new policy.
- `thread_goal_actions.rs:75-79` and `:103-104` — `goal_status_for_plan_mode(requested, plan_mode_active)` runs in the TUI before issuing the RPC. This is the right UX, but it means **any non-TUI client of the app-server can still set `ThreadGoalStatus::Active` while a thread is in plan mode**. The check is no longer enforced server-side. If that's intentional (TUI-only feature for now), say so in the PR body; otherwise the policy should also live in the app-server `thread_goal_set` handler.
- `goals.rs:954-964` — `pause_active_thread_goal_for_interrupt` loses its plan-mode short-circuit. Net behavior: an interrupt while in plan mode now pauses the goal even though plan mode was already supposed to keep it paused. Idempotent, but worth confirming the interrupt path doesn't double-record a pause event in the goal history table.
- The deletion of `activate_paused_thread_goal_after_resume()` (lines 998-1097 in the old file, fully removed) and the `ThreadResumed` event plumbing is the real semantic change. Resumed threads now stay paused — this matches the PR title and is the safer default. Test `resumed_thread_keeps_paused_goal_paused` (renamed from `resumed_thread_activates_paused_goal_and_continues_on_request`) at `thread_resume.rs:739` covers it.
- `thread_resume.rs:78-160` — `thread_resume_emits_paused_goal_snapshot_without_continuation` asserts `"status": "paused"` and `"paused goals should not continue automatically on thread resume"`. Good explicit assertion. The plan-mode case at `:252-273` constructs `let plan_mode = CollaborationMode { ... }; materialize_thread(&mut mcp, &thread_id, Some(plan_mode)).await?` and asserts `"paused"`. Both look well-targeted.
- `app/thread_goal_actions.rs:90-91` — `Ok(response) => self.show_thread_goal_updated(&response.goal)` — the previous code showed both `goal_status_label` and `goal_usage_summary`. Verify `show_thread_goal_updated` renders both pieces (the helper isn't shown in the patch context, but its absence would be a regression).
- `footer.rs:874-898` — `goal_paused_in_plan_mode_line(usage: Option<&str>)` builds either `"Goal paused in Plan mode (usage)"` or `"Goal paused in Plan mode"`. Reads cleanly. Small nit: the phrasing "in Plan mode" capitalizes Plan but the rest of the codebase uses `ModeKind::Plan` lowercased in the goal-menu snapshots — verify both call sites agree.
- Three new insta snapshot files (`goal_menu_active_plan_mode`, `goal_menu_paused_plan_mode`, `status_line_goal_active_plan_mode_footer`) are added, total 35 lines. These pin the visual contract; reviewers should `cargo insta review` against the actual rendering before approving.
- Net deletion of 134 lines from `core/src/goals.rs` is the right shape — moving policy out of the runtime into the call site simplifies the runtime contract. Just don't lose the policy on the way to the next non-TUI consumer.

## Verdict: `merge-after-nits`

The behavior change is correct and well-tested. Two things to fix before merge: (1) update the `goal_runtime_apply` doc comment in `goals.rs:507` to explicitly state that plan-mode pause policy now lives at the call site, not in the runtime; (2) decide whether non-TUI app-server clients should be allowed to set `Active` in plan mode, and either enforce server-side or document the TUI-only scope. Optional: a regression test that asserts `pause_active_thread_goal_for_interrupt` is idempotent against an already-paused goal.
