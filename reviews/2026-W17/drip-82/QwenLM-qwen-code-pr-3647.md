# QwenLM/qwen-code PR #3647 — fix(cli): keep sticky todo panel compact

- Repo: QwenLM/qwen-code
- PR: #3647
- Head SHA: `50322c7e`
- Author: @shenyankm
- Diff: +202/-34 across 8 files

## What changed

Adds a `maxVisibleItems?: number` prop to `<StickyTodoList>` plus a new exported `getStickyTodoMaxVisibleItems(terminalHeight)` helper that derives the cap as `floor(height/TERMINAL_ROWS_PER_VISIBLE_TODO)`, clamped to `[MIN_VISIBLE_TODOS=1, DEFAULT_MAX_VISIBLE_TODOS=5]`. Long-content todos are now single-line truncated and overflow is summarized with a literal `... and N more` trailer instead of letting the panel wrap into the chat scrollback.

## Specific observations

- The clamp helper at `packages/cli/src/ui/components/StickyTodoList.tsx:33-43` (`clampVisibleTodoCount`) correctly handles the three edge cases: non-finite (returns default 5), below floor (raises to 1), above ceiling (caps at 5). The `Math.floor(value)` step before clamping is the right call to defend against fractional terminal-row math from upstream callers. Constants `DEFAULT_MAX_VISIBLE_TODOS = 5` / `MIN_VISIBLE_TODOS = 1` / `TERMINAL_ROWS_PER_VISIBLE_TODO = 5` at lines 28-30 are reasonable defaults but should be commented with the rationale (why 5 rows per todo? — looks like 1 status line + 1 wrapped content line + 1 spacer × ~1.7).
- Test coverage at `StickyTodoList.test.tsx:60-105` is solid: the "compact" assertion verifies `expect(output).toContain('... and 2 more')` plus `expect(lines).toHaveLength(6)` for tight overflow control, and the viewport-aware test at `:104` exercises the helper at heights 8 (→1), 15 (→3), 80 (→5) covering both clamp edges and an interior case. Missing: an `Infinity`/`NaN` case to lock down the `!Number.isFinite(value)` branch, and a `maxVisibleItems={0}` explicit-prop case to confirm the floor wins over caller intent.
- The `lastFrame()?.split('\n').filter(Boolean)` pattern at the test is brittle against any future cosmetic blank-line addition to the panel render — consider asserting on a structural marker (e.g., the `... and N more` line index) rather than total line count.

## Verdict

`merge-after-nits`

## Rationale

Clean, well-tested fix for a real visual bug. Add the `Infinity` / `0` edge tests, document the magic numbers, and consider a structural rather than line-count assertion. Otherwise straightforwardly mergeable.
