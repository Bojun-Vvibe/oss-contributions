# openai/codex PR #19929 — TUI: use cumulative turn duration for worked-for separator

- **PR**: https://github.com/openai/codex/pull/19929
- **Author**: @etraut-openai (Eric Traut)
- **Head SHA**: `579adb183c780f75492f46bd56be07647d9ee36c`
- **Size**: +121 / −58 across 7 files
- **Closes**: #19814

## Summary

Replaces the per-separator incremental timing scheme (legacy of #9599 from before turns had protocol-native `duration_ms`) with a direct read of the new `TurnCompleteEvent.duration_ms` / `TurnCompleted.turn.duration_ms` field, with a fallback to the existing status-indicator elapsed-time only when the protocol value is unavailable. Mid-turn dividers between assistant text segments lose their `Worked for ...` clock and become plain visual dividers. Removes the now-unused `last_separator_elapsed_secs` state and `worked_elapsed_from()` helper.

## Verdict: `merge-as-is`

Right cleanup at the right time. The original incremental-timer was correctly described in `chatwidget.rs` (now-deleted) comments as a workaround for the pre-`duration_ms` era, so removing it once the upstream signal is reliable is the textbook lifecycle. The new code is shorter, has fewer pieces of mutable per-turn state, and the snapshot test pins the load-bearing UI string. The fallback-to-status-indicator at `chatwidget.rs:2814-2818` is the right amount of conservatism for the case where a session was started before the `duration_ms` field existed in the protocol.

## Specific references

- `codex-rs/tui/src/chatwidget.rs:2755-2761` — `on_task_complete` signature gains `duration_ms: Option<i64>` between `last_agent_message` and `from_replay`. Three positional `Option<*>` params now (one of them an `i64`), which is the kind of call-site that benefits from named-args at the call site or a dedicated struct, but the test sites at `tests/exec_flow.rs:601-603` and `:617-619` use the `/*duration_ms*/ None` comment convention which is the right discipline for this codebase. Acceptable.
- `codex-rs/tui/src/chatwidget.rs:2805-2818` — load-bearing change: `show_work_separator` now also fires when `runtime_metrics.is_some()` (previously only when `needs_final_message_separator && had_work_activity`). This is correct — runtime metrics are themselves the signal that work happened, and prior code could lose the "Worked for" clock on a turn that finished without an intervening exec output. The conversion ladder `duration_ms -> u64::try_from -> /1_000` correctly drops the milliseconds portion, and `or_else` to status widget is the right fallback shape.
- `codex-rs/tui/src/chatwidget.rs:2735` — `had_work_activity = false` is now reset alongside the other per-turn flags in the turn-start handler instead of being lazily reset after separator emission at the deleted `:5051`. Cleaner: turn-start is the natural barrier for per-turn state, not "next time we draw a separator."
- `codex-rs/tui/src/chatwidget.rs:5050-5053` — mid-turn separator now constructs `FinalMessageSeparator::new(/*elapsed_seconds*/ None, /*runtime_metrics*/ None)`. This loses the per-chunk timer between two assistant message parts. PR body explicitly calls this out as intentional ("Keep mid-turn separators before later assistant text as plain visual dividers instead of clocked `Worked for ...` separators"). Right call — the prior behavior showed two `Worked for X` separators in the same turn which double-counted intuitively.
- `codex-rs/tui/src/chatwidget.rs:7253-7259` — `TurnStatus::Completed` arm at the v2 protocol layer correctly threads `notification.turn.duration_ms` (already `Option<i64>` from the protocol schema) into `on_task_complete`. The `EventMsg::TurnComplete` arm at `:7610-7615` does the same for the legacy event path. Both surfaces now feed the same canonical timing, which is the right symmetry.
- `codex-rs/tui/src/chatwidget/snapshots/codex_tui__chatwidget__tests__final_worked_for_uses_cumulative_turn_duration.snap:1-12` — the asserted output literally pins `─ Worked for 2m 05s ─` after a single exec + final answer, which is exactly the regression-anchor shape. The test at `exec_flow.rs:716-770` (per the diff hunk) constructs a turn with a `duration_ms` of `125_000` (2m 5s) and asserts the rendered separator. Strong regression protection.

## Nits (not blocking)

1. The deleted comment block at the original `chatwidget.rs:1014-1019` documenting the workaround was historically useful context — consider preserving it in a commit message or a `// removed in #19929: ...` breadcrumb so future archaeologists can find the rationale.
2. `Option<i64>` → `u64` via `try_from().ok().map(/1_000)` silently swallows negative durations. In practice the server should never emit a negative `duration_ms`, but a `tracing::warn!` on the negative-conversion path would catch a server-side regression earlier than "the UI doesn't show a duration."
3. The test name `final_worked_for_uses_cumulative_turn_duration_snapshot` is excellent; consider also adding a cell for "duration_ms is None, falls back to status widget" so the fallback path has an explicit pin too.
