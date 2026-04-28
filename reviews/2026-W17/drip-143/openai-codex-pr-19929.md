# openai/codex#19929 — TUI: use cumulative turn duration for worked-for separator

- Head SHA: `579adb1`
- Author: etraut-openai
- Files: 7 / +121 / −58
- Closes #19814

## Summary

Replaces the per-separator incremental-elapsed-seconds workaround (a leftover from #9599 written before turn-completion duration was protocol-native) with the protocol-provided `duration_ms` field on `TurnCompleteEvent`/`TurnCompleted`. Threads `duration_ms: Option<i64>` into `ChatWidget::on_task_complete` from both legacy event handling and the app-server notification path. Removes the now-dead `last_separator_elapsed_secs` state field, the `worked_elapsed_from(current)` helper, and demotes mid-turn separators-before-later-assistant-text from clocked `Worked for ...` to plain visual dividers.

## Specific observations

- Signature flip at `codex-rs/tui/src/chatwidget.rs:2758-2763` — `on_task_complete(&mut self, last_agent_message: Option<String>, duration_ms: Option<i64>, from_replay: bool)` — three callers updated: legacy `EventMsg::TurnComplete(TurnCompleteEvent { duration_ms, .. })` at `:7611-7615`, app-server `TurnStatus::Completed` at `:7256-7260`, and two test callsites in `tests/exec_flow.rs:601` / `:617` updated with explicit `/*duration_ms*/ None`. Symmetric.
- Load-bearing precedence at `:2808-2818` — `duration_ms.and_then(|d| u64::try_from(d).ok()).map(|d| d / 1_000).or_else(|| status_widget.elapsed_seconds())` — protocol-provided wins, status-widget fallback only when protocol value absent or negative (the `try_from(i64) → u64` discriminator silently drops negative values, which is correct because a negative duration is nonsense; but no test pins this behavior).
- Reset of `had_work_activity` moved from "after emitting separator" to `:2732` (turn-start reset block alongside `saw_copy_source_this_turn`/`saw_plan_update_this_turn`). This is the right place — the previous "reset on emit" path had a subtle bug where an interrupted turn that never emitted a separator would carry `had_work_activity = true` into the next turn. The new reset-on-start closes that.
- Show-condition tightened at `:2808` from `needs_final_message_separator && had_work_activity` to `had_work_activity && (needs_final_message_separator || runtime_metrics.is_some())` — explicit `runtime_metrics.is_some()` arm preserves the runtime-metrics-only display case where the user has metrics enabled but no final-separator boundary was hit. This is the discrimination the old code was getting wrong.
- Snapshot test `final_worked_for_uses_cumulative_turn_duration_snapshot` at `tests/exec_flow.rs:719+` with the load-bearing `─ Worked for 2m 05s ─` line in the snap file at `chatwidget__tests__final_worked_for_uses_cumulative_turn_duration.snap:11` — pins both that cumulative duration is consumed AND that the `2m 05s` formatter (from the `human_seconds` helper) is invoked. The snap file is small (12 lines) and well-scoped, not a 200-line render-of-everything snapshot.
- Mid-turn separator at `:5050-5051` is now `FinalMessageSeparator::new(/*elapsed_seconds*/ None, /*runtime_metrics*/ None)` — strips the clocked time deliberately. The PR body explains this is intentional (mid-turn dividers should be plain visual breaks, not falsely-precise per-chunk timings), but the in-code change has only the `/*elapsed_seconds*/ None` comment as documentation — a future "consistency" patch could re-add the elapsed-seconds threading without realizing it's a deliberate UX choice.

## Verdict

`merge-after-nits`

## Rationale

Right shape: replaces a workaround that's no longer needed with the protocol-native field, deletes 58 lines of state management as a result, and the snapshot test pins the user-visible output. Correctness boundary on the `try_from(i64) → u64 → / 1_000` chain is good (negative durations silently fall through to the status-widget fallback rather than wrapping or panicking). Nits: (1) add a test cell for the negative-`duration_ms` fallback to status-widget — silent drop is the right behavior but should be pinned; (2) add a test cell for `duration_ms < 1000` (sub-second turn) where `/ 1_000` produces `0`, which probably renders as `Worked for 0s` and may look broken — verify or floor at 1; (3) add an inline comment at `:5050` naming why mid-turn separators deliberately don't show elapsed time, citing this PR or the linked issue, to prevent regression-by-cleanup.

