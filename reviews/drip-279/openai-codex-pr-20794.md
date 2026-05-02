# openai/codex PR #20794 — feat(tui): add keymap debug inspector

- URL: https://github.com/openai/codex/pull/20794
- Head SHA: `26a6252e8ba9f4009f9fce5b118b6cc8c1da0b00`
- Files touched: 20 (+583 / −19)
- Verdict: **merge-after-nits**

## Summary
Adds `/keymap debug` (and a Debug tab in the existing keymap picker) that opens
a TUI overlay showing exactly which key events the runtime is receiving — useful
for diagnosing terminals that mangle modifier keys. Introduces a new
`AppEvent::OpenKeymapDebug`, a `next_frame_delay` hook on `BottomPaneView`, and
a chatwidget entry point `open_keymap_debug`.

## Cited concerns

1. `codex-rs/tui/src/bottom_pane/mod.rs:716-727` (post-patch):
   `pre_draw_tick_at` now calls `schedule_active_view_frame()` on every tick.
   The new `schedule_active_view_frame` itself queries `active_view()` and
   reschedules a redraw at `delay`. If a view returns `Some(Duration::ZERO)`,
   this becomes an unbounded redraw loop. Default impl returns `None`
   (`bottom_pane_view.rs:135-137`), but worth either documenting "must be > 0"
   or clamping to a minimum (e.g. 16 ms) inside `request_redraw_in`.

2. `bottom_pane/bottom_pane_view.rs:134-137`: the new trait method has a
   default `None` body and no rustdoc beyond one line. Given this is a new
   public-ish surface for every BottomPaneView impl, expand the doc to
   describe the contract: returned delay is *minimum* time before next frame,
   `None` means "no time-based redraw needed", caller may coalesce with other
   redraw triggers.

3. `app/event_dispatch.rs:1925-1928`: `OpenKeymapDebug` dispatches to
   `chat_widget.open_keymap_debug(&self.keymap)`. No guard against opening
   the inspector while another modal (e.g. approval dialog) is already on the
   view stack. Probably fine because `push_view` stacks correctly, but a
   regression test that exercises "open approval → /keymap debug → Esc → back
   to approval" would be valuable for the LIFO behavior.

4. PR is +583 with 20 files; would benefit from a screenshot or asciicast in
   the PR body to make reviewer evaluation faster.

## Verdict
**merge-after-nits** — useful diagnostic feature, clean event-dispatch
extension. Address the redraw-loop risk on `next_frame_delay(Some(0))` and
expand rustdoc, then ship.
