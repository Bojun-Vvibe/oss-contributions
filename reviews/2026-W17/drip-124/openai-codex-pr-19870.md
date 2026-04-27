# openai/codex #19870 — feat(tui): source-back assistant streaming in TUI

- **PR**: [openai/codex#19870](https://github.com/openai/codex/pull/19870)
- **Head SHA**: `cbd37741`
- **Stats**: +229/-258 across 10 files in `codex-rs/tui/`

## Summary

Refactors how live assistant streaming renders in the TUI. Previously,
incoming `AgentMessageDelta` events queued transient
`AgentMessageCell` instances into the transcript and a separate
`ConsolidateAgentMessage { source, cwd }` event (emitted on stream
finalization) walked back through the transcript to splice the
trailing run of streaming cells into one source-backed
`AgentMarkdownCell`. The new design renders deltas through a single
*active* source-backed markdown cell from the first delta, then commits
that same cell to history on stream completion — eliminating the
queue-based intermediate state and the consolidation-event walk-back.
The `AppEvent::ConsolidateAgentMessage` variant is removed entirely
(`app_event.rs:392-404`).

## Specific findings

- `app_event.rs:392-404` (deleted) — `AppEvent::ConsolidateAgentMessage
  { source, cwd }` is gone. This is the load-bearing protocol change:
  any external app-event consumer (none in-tree) breaks at compile
  time. Correct cadence — leaving the variant as a deprecated no-op
  would invite drift.
- `app/event_dispatch.rs:206-230` (deleted) — the entire
  `AppEvent::ConsolidateAgentMessage` handler arm is removed. Removed
  code includes the `trailing_run_start::<AgentMessageCell>` walk-back
  + `transcript_cells.splice(start..end, ...)` consolidation +
  `Overlay::Transcript.consolidate_cells` mirror. The new design
  doesn't need this because there's never a "run of streaming cells"
  to coalesce — there's always a single active cell.
- `app/resize_reflow.rs:467-477` —
  `should_mark_reflow_as_stream_time` simplified from four conditions
  (active agent stream OR active plan stream OR trailing
  `AgentMessageCell` run OR trailing `ProposedPlanStreamCell` run) to
  two (active plan stream OR trailing `ProposedPlanStreamCell`
  run). The agent-stream conditions are gone because there is no
  longer a "trailing transient agent cells" state to detect — the
  active cell is committed atomically on `ItemCompleted`.
- `streaming/controller.rs:798-810` — controller now exposes
  `active_cell()` returning a fresh `AgentMarkdownCell` constructed
  from the controller's accumulated `markdown_source` + `cwd`. The
  resize-reflow path at `:806-810` returns
  `Some(Box::new(AgentMarkdownCell::new(...)))` so on a width change
  the active cell can re-render from source. This is the core of
  "source-backed" streaming — the rendered text is always derived
  from the markdown source, never accumulated as pre-rendered
  display text, so resize during stream produces correctly-wrapped
  output instead of fixed-width-relict lines.
- `app/tests.rs:3914-3978` —
  `finalized_assistant_stream_is_source_backed_for_resize_reflow`
  test: feeds an `AgentMessageDelta`, asserts no
  `InsertHistoryCell` is emitted yet (deltas update the active cell
  in place), then feeds an `ItemCompleted` with
  `AgentMessageItem::content`, expects exactly one
  `InsertHistoryCell` whose underlying type is `AgentMarkdownCell`,
  and renders at width 100 vs 28 to assert the cell re-flows
  (`narrow.lines.len() > wide.lines.len()`). Critically asserts the
  draft text from the delta is *not* present in the final render
  (`!normalized_narrow_text.contains("draft text")`) — pinning that
  the final commit replaces the draft, doesn't append to it.
- `app/tests.rs:371` — companion test
  `assistant_delta_updates_source_backed_active_cell_without_history_insert`
  pins the no-history-insert-during-deltas invariant separately so a
  regression that incorrectly emits per-delta `InsertHistoryCell`
  events fails this test rather than the more complex
  resize-reflow test.

## Nits / what's missing

- The `AppEvent::ConsolidateAgentMessage` removal is a v2 protocol
  break for external app-event consumers (the variant is `pub(crate)`
  on the enum but the `AppEvent` enum itself is part of the TUI's
  public-ish surface). PR body / commit message doesn't carry a
  `BREAKING:` callout. For an in-tree-only TUI the impact is zero;
  for an out-of-tree fork shipping a custom event handler it's a
  silent compile break. Worth one line in the PR description.
- `controller.rs:798` — `active_cell()` constructs a *new*
  `AgentMarkdownCell` on every call by cloning `markdown_source`. For
  a long stream (thousands of deltas) this is O(N²) total in markdown
  bytes. The actual render path likely doesn't call `active_cell()`
  per-delta — but the method's contract should be commented to
  prevent a future "render every delta through `active_cell()`"
  optimization that would actually pessimize the common case.
- The two stream-time conditions removed from
  `should_mark_reflow_as_stream_time` carried a paragraph-long
  explanatory doc comment about the
  "narrow-window-after-controller-stops-but-before-consolidation"
  problem. The new doc comment correctly drops the agent-stream
  reference but the *plan*-stream branch still uses the same
  pattern. Worth a one-line comment that "this is intentional asymmetry
  — assistant streaming doesn't need this guard because the active
  cell commits atomically on completion, but plan streaming still
  uses the queue-and-consolidate pattern".

## Verdict

**merge-after-nits** — substantial simplification of the streaming
render path that removes a class of race conditions
(transient-cell-run-walk-back-during-resize) by construction. Test
coverage is well-targeted (separate tests for the no-history-insert
invariant and the resize-reflow-on-final invariant). Three nits:
(1) `BREAKING:` callout for the dropped `AppEvent` variant, (2)
documented contract for `controller.active_cell()` allocation cost,
(3) one-line comment naming the deliberate
plan-stream-vs-assistant-stream pattern asymmetry in
`should_mark_reflow_as_stream_time`. None blocking.
