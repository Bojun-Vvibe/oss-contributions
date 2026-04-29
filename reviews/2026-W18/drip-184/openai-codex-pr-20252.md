---
pr: openai/codex#20252
sha: 2f948bac0377841311130a893989c1a7953d4fde
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# feat(tui): render responsive Markdown tables in TUI

URL: https://github.com/openai/codex/pull/20252
Files: `codex-rs/tui/src/markdown_render.rs`,
`codex-rs/tui/src/markdown_render/table.rs` (new ~530 LOC),
`codex-rs/tui/src/markdown_render_tests.rs`,
`codex-rs/tui/src/chatwidget.rs`, `codex-rs/tui/src/streaming/controller.rs`,
`codex-rs/tui/src/streaming/mod.rs`, `codex-rs/tui/src/app/tests.rs`,
plus a snapshot
Diff: 1671+/52-

## Context

The TUI's Markdown renderer treated tables as inline text, so a model
emitting a real GitHub-flavored Markdown table got a wrapped-prose mess
in the TUI even when the same content rendered cleanly on the web. Worse,
finalized table cells already in scrollback didn't survive a terminal
resize because the renderer only stored the post-render `Line` array,
not the source markdown — there was nothing to re-flow against the new
width. This PR adds a real table renderer with two layout strategies,
and changes the streaming pipeline so the in-progress block stays as a
"live tail" pinned to current width while only stable blocks get
committed to scrollback as raw markdown that can be re-rendered on
resize.

## What's good

- New `markdown_render/table.rs:523` `render_table_lines(rows, width)`
  is the single public entry point with two strategies: the box layout
  (`render_box_table`, `:806`) for terminals wide enough to fit columns
  with a sensible minimum, and `render_vertical_table` (`:1066`) which
  flips into a record-per-row "field: value" layout once the box layout
  would produce columns narrower than the per-cell minimum. The
  threshold pivot is the load-bearing readability decision — degrading
  to vertical-stacked is far better than letting the box render columns
  too narrow to read.
- `render_table_row` at `:826` does the per-cell wrap and pad bookkeeping
  that the rest of the renderer assumes — width math is centralised here
  so border-character contributions, padding, and column-min are all
  computed in one place rather than scattered across the box/vertical
  paths.
- Streaming pipeline change in `chatwidget.rs:2039-2045` adds an explicit
  `self.active_cell = None; self.bump_active_cell_revision();` *before*
  taking the stream controller in `flush_answer_stream_with_separator`,
  so the live tail is detached from the active cell exactly when the
  finalized cell is about to be moved to scrollback. The new
  `sync_agent_stream_active_tail` / `sync_plan_stream_active_tail` calls
  at `:4781-4787,4858` re-attach the active tail after each batch of
  emitted cells — this is what makes the "stable blocks finalized,
  current block live" model work.
- The `flush_active_cell` short-circuit at `:5919-5921` (now gated on
  `self.stream_controller.is_none() && self.plan_stream_controller.is_none()`)
  prevents the active tail from being prematurely flushed when a
  non-stream history insert (exec output, patch status) lands mid-stream,
  which is the actual root cause of the "table jumps to scrollback
  half-rendered" class of bugs.
- New test at `app/tests.rs:3917-3978`
  `table_resize_lifecycle_reflow_preserves_scrollback_and_rerenders_width`
  is the right shape: builds a transcript with `BEFORE_TABLE_SENTINEL`,
  a finalized markdown-table cell, and `AFTER_TABLE_SENTINEL`, then
  re-renders at width 44 and width 96. Asserts (a) sentinels appear
  exactly once at both widths (no duplication on reflow), (b) the box
  shape (`'┌'`) survives in the finalized scrollback at both widths,
  (c) narrow-mode table-row lines respect `line.width() <= 44` (no
  horizontal overflow), (d) `narrow_text.len() > wide_text.len()` (the
  reflow actually changed the wrap, not just re-rendered the same
  layout) — that fourth assertion is the one that catches "scrollback
  re-render is a no-op" regressions. Use of `unicode_width::WidthStr`
  for the width predicate is correct (handles wide CJK glyphs).
- The complex-snapshot file
  `markdown_render__markdown_render_tests__markdown_render_complex_snapshot.snap`
  is updated atomically with the renderer change so a regression is
  one `cargo insta` away from being visible.

## Nits

- `render_table_lines` at `:523` takes `width: Option<usize>` — the
  `None` case isn't documented in the function signature; clarify
  whether `None` means "uncapped, render at natural width" vs "use
  default" so callers don't have to read the body to know.
- The vertical-stacked fallback (`:1066`) is the right escape hatch but
  doesn't appear to be exercised in the new app test; add a width=20 or
  width=24 case asserting that `render_table_lines` no longer emits
  `'┌'` and instead emits the field-label form, so the
  layout-strategy-pivot threshold gets a regression fence.
- `chatwidget.rs:2039-2042` adds the `had_stream_controller` guard
  before mutating `active_cell` / `bump_active_cell_revision` — fine,
  but the symmetric path in `:2666-2670` for the plan stream does the
  guard inline (`if self.plan_stream_controller.is_some()`). Pick one
  pattern; the inconsistency will bait the next refactor into reading
  one site as the canonical shape and breaking the other.
- The tail-sync calls at `:4781-4787` only run when `outcome.cells` is
  non-empty (`emitted_cells`). If a cell-emission burst is empty but
  the controller's internal state advanced (e.g., delta absorbed but
  no cell finalized yet), the tail won't refresh — probably correct
  given current controller behaviour but worth a one-line comment so
  the next person doesn't add a "fix" that re-syncs every tick.
- 1671 LOC added with ~530 in a single new module is a lot to review
  in one PR — splitting into (1) "extract markdown source preservation
  + reflow plumbing" and (2) "add table renderer module" would make
  bisecting a future regression to either the layout engine or the
  reflow pipeline trivial. Not blocking on a feature PR, but worth
  noting for next time.
- No keyboard / mouse integration with table cells (e.g., copy-table-
  as-tsv) — out of scope for this PR but worth a follow-up issue
  filed alongside the merge so the "we have real tables now" platform
  is actually reachable from the UX.

## Verdict reasoning

Substantive, well-tested feature with the right architectural choice
(preserve raw markdown for stable blocks, keep the in-progress block as
a live width-sensitive tail). The lifecycle test is the kind that
would have caught the prior regression class — sentinels-around-table
plus a length comparison across widths is exactly the shape needed to
catch "reflow is silently a no-op". The two-strategy renderer (box vs
vertical) is the right fallback for narrow terminals. Nits are about
documenting the `None` width case, exercising the vertical fallback in
app tests, harmonising the active-cell-revision guard pattern, and
splitting the 1671-line PR for future bisectability — none blocking.
