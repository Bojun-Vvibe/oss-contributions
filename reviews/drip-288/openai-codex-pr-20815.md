# openai/codex PR #20815 — Speed up /side parent restore replay

- Head SHA: `6acf705441af2349e4d00d9cfce9e1cdf5107857`
- URL: https://github.com/openai/codex/pull/20815
- Size: +77 / -0, 6 files (`codex-rs/tui/src/{app.rs,app/event_dispatch.rs,app/resize_reflow.rs,app/tests.rs,app/thread_routing.rs,app_event.rs}`)
- Verdict: **merge-after-nits**

## What changes

When returning from a `/side` conversation, the parent thread is
restored by replaying its full snapshot through `handle_thread_event_replay`.
For long parent threads this writes every transcript row to terminal
scrollback, even though `terminal_resize_reflow.max_rows` already caps
what the terminal will retain. This PR adds a new `BeginThreadSwitchHistoryReplayBuffer`
event that flips a `render_from_transcript_tail: bool` flag on the
existing `InitialHistoryReplayBuffer`. While the flag is set,
`buffer_resume_replay_lines_for_history` early-returns (no per-cell
rendering); on `EndInitialHistoryReplayBuffer` the buffer flushes a
single tail render via `render_transcript_lines_for_reflow(width)`.

## What looks good

- The reuse of the existing `InitialHistoryReplayBuffer` struct +
  `EndInitialHistoryReplayBuffer` event (thread_routing.rs line ~1273)
  is the right call — the resume-replay path already had the exact
  buffer-then-flush plumbing this needs, no new channel required.
- The activation gate in `begin_thread_switch_history_replay_buffer`
  (resize_reflow.rs line ~123) checks all three preconditions
  (`terminal_resize_reflow_enabled` && `resize_reflow_max_rows().is_some()`
  && `self.overlay.is_none()`) before opting in. Without the row cap
  the optimization would *lose* rows the terminal would otherwise
  retain — the gate prevents that.
- The two new tests (app/tests.rs line ~3989, ~4011) cover both the
  enabled (`Limit(3)`) and disabled (`TerminalResizeReflowMaxRows::Disabled`)
  paths and assert on `render_from_transcript_tail` directly. Good
  coverage of the gate.
- `buffer_resume_replay_lines_for_history` early-return
  (resize_reflow.rs line ~150) is gated by
  `is_some_and(|buffer| buffer.render_from_transcript_tail)` so it
  only short-circuits in the new mode — the existing initial-replay
  path is untouched.

## Nits

1. `should_buffer_replay = self.terminal_resize_reflow_enabled() &&
   (!snapshot.turns.is_empty() || !snapshot.events.is_empty())`
   (thread_routing.rs line ~1241) doesn't mirror the row-cap check
   that `begin_thread_switch_history_replay_buffer` enforces. So the
   `Begin` event always fires when reflow is enabled, even if there's
   no `max_rows`. The handler will no-op (per the gate at
   resize_reflow.rs line ~123), but the `End` event (line ~1273) still
   fires unconditionally and flushes nothing. That's *correct* but
   emits two extra events per thread switch on the no-cap config —
   collapse the gate at the call site so the events don't fire when
   they can't do anything.
2. The new event variant docstring (app_event.rs line ~488) is good
   but doesn't note the flush is opt-in only when `max_rows` is
   `Some`. A reader of `app_event.rs` alone can't tell when this
   event matters. Cross-link to `begin_thread_switch_history_replay_buffer`.
3. `buffer.render_from_transcript_tail = true` is set in
   `begin_thread_switch_history_replay_buffer` but
   `flush_initial_resume_replay_buffer` (resize_reflow.rs line ~140)
   uses the resume-style "if retained_lines is empty" branch to
   trigger the tail render. That coupling means: if a stray
   `buffer_resume_replay_lines_for_history` *did* push something into
   `retained_lines` while `render_from_transcript_tail` was set, the
   tail render would be skipped and the per-line render path would
   run instead. The early-return at line ~150 prevents this, but a
   `debug_assert!(buffer.retained_lines.is_empty())` inside the
   `render_from_transcript_tail` arm at line ~152 would catch a
   future regression.

## Risk

Low. The change is invisible unless `terminal_resize_reflow.max_rows`
is `Limit(N)` *and* a `/side` parent restore happens. In that case the
output is functionally identical (same tail rows in scrollback) but
arrives via a single `insert_history_lines` call instead of N
per-cell writes. Worst case if the gate logic is wrong: replay output
is empty until the next render — visually obvious in QA, not a data
integrity bug.
