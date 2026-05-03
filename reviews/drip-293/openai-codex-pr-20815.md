# openai/codex PR #20815 — Speed up /side parent restore replay

- **Repo:** openai/codex
- **PR:** #20815
- **Head SHA:** `6acf705441af2349e4d00d9cfce9e1cdf5107857`
- **Author:** etraut-openai
- **Title:** Speed up /side parent restore replay
- **Diff size:** +77 / -0 across ~5 files (`tui/src/app.rs`, `app/event_dispatch.rs`, `app/resize_reflow.rs`, `app/tests.rs`, `app/thread_routing.rs`, `app_event.rs`)
- **Drip:** drip-293

## Files changed

- `tui/src/app.rs` (+1) — adds `render_from_transcript_tail: bool` field to `InitialHistoryReplayBuffer` struct (line ~426).
- `tui/src/app/event_dispatch.rs` (+3) — wires new `AppEvent::BeginThreadSwitchHistoryReplayBuffer` to a new method.
- `tui/src/app/resize_reflow.rs` (+~26) — adds `begin_thread_switch_history_replay_buffer()` (lines ~127-140) and an early-out branch in `flush_initial_history_replay_buffer` (lines ~150-156) that, when the buffer is empty *and* `render_from_transcript_tail` is set, calls `render_transcript_lines_for_reflow(width)` and inserts only the rows the terminal would retain. Also short-circuits per-cell `display_lines_for_history_insert` (lines ~170-176) while the tail buffer is active.
- `tui/src/app/tests.rs` (+27) — two new tokio tests asserting the buffer is set with `render_from_transcript_tail = true` when reflow + row cap are present, and `None` when no row cap.
- `tui/src/app/thread_routing.rs` (+10) — wraps the thread-switch replay in `BeginThreadSwitchHistoryReplayBuffer` / `EndInitialHistoryReplayBuffer` events when reflow is enabled and the snapshot has turns/events.
- `tui/src/app_event.rs` (+4) — adds the new event variant with doc comment.

## Specific observations

- `app/resize_reflow.rs:147-156` — the empty-buffer + tail-mode early-out is the load-bearing optimization: instead of formatting every cell, it renders the transcript tail once at flush time. Correct. But note the asymmetry: `EndInitialHistoryReplayBuffer` is reused (per `thread_routing.rs:1272`) instead of a dedicated `EndThreadSwitchHistoryReplayBuffer`. That works because `flush_initial_history_replay_buffer` polymorphically branches on `render_from_transcript_tail`, but it makes the event name misleading. Either rename the event to `EndHistoryReplayBuffer` or add a comment in `app_event.rs` noting the dual use.
- `app/resize_reflow.rs:170-176` — the early-return inside `buffer_initial_history_replay_row` (suppressing per-cell formatting while tail-mode is active) is the right move, but it depends on `is_some_and(|buffer| buffer.render_from_transcript_tail)` being checked on *every* cell. For a large replay this is a cheap branch, but worth confirming via `cargo bench` / a perf note in the PR description.
- `app/thread_routing.rs:1241-1245` — `should_buffer_replay = ... && (!snapshot.turns.is_empty() || !snapshot.events.is_empty())` correctly avoids the optimization on no-op snapshots, but consider also gating on `resize_reflow_max_rows().is_some()` here (matching `begin_thread_switch_history_replay_buffer`'s own guard) so the `Begin`/`End` event pair isn't emitted when the receiver will no-op. Currently it's a wasted event round-trip.
- `app/tests.rs:3989-4015` — both new tests are tight and exactly hit the two contract conditions (cap present → buffer set + tail-mode true; cap absent → buffer None). Good. Missing: a test covering the *flush* path (i.e. that `render_transcript_lines_for_reflow` is actually invoked and the rows match expected tail). Worth adding even as an integration-style assertion against `tui.insert_history_lines` calls.
- `app_event.rs:488-491` — doc comment on `BeginThreadSwitchHistoryReplayBuffer` is clear; mirrors the existing `BeginInitialHistoryReplayBuffer` style.

## Verdict: `merge-after-nits`

Good targeted optimization that avoids per-cell formatting on thread switches when a row cap is present, with structurally sound state-machine integration. Asks: (a) clarify the dual use of `EndInitialHistoryReplayBuffer` (rename or comment), (b) gate `Begin`/`End` event emission in `thread_routing.rs` on the same `resize_reflow_max_rows().is_some()` predicate the receiver checks, (c) one flush-path test. Optimization is real; just polish.
