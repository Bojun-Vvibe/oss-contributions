# QwenLM/qwen-code PR #3663 — fix(cli): stop streaming clear storms in main TUI

- Link: https://github.com/QwenLM/qwen-code/pull/3663
- Head SHA: `231df265`
- Size: +808 / -21 across ~6 files (includes 170-line design doc + 276-line ANSI-capture integration test + 90-line slicer + shell-output dedup tweak)

## Summary

Supplemental TUI flicker fix after #3591. Targets one specific reproducible class still open: long assistant streaming text taller than the terminal triggers Ink 6.2.3 to emit `clearTerminal` (`ESC[2J ESC[3J ESC[H`) and replay all `<Static>` output before each new dynamic frame, producing a visible full-screen flash and banner replay on every chunk. The fix bounds the *live pending* viewport by visual height in `ConversationMessages.tsx` while preserving the full final-transcript markdown render after streaming completes; a separate change in `shellExecutionService.ts` suppresses resize-only soft-wrap reflows so narrow-width wraps don't masquerade as fresh shell output.

## Specific-line citations

- `docs/design/tui-optimization/streaming-clear-storm.md:1-170` (new) is the canonical write-up: source-level root cause is the chain `MainContent.tsx` → `<Static>` + dynamic pending → `HistoryItemDisplay` → `ConversationMessages` → `MarkdownDisplay` where pending paragraphs/single-line text could wrap past `stdout.rows` and trigger Ink's full-screen-clear safety. The doc explicitly scopes itself to streaming, **not** "every TUI flicker" — resize, compact toggles, deliberate static-output replacement still need separate validation. That kind of bounded scope statement is unusually disciplined for a flicker fix.
- `ConversationMessages.tsx:84-150` introduces `slicePendingTextForHeight(text, maxHeight, maxWidth)` — uses `toCodePoints(text)` for correct grapheme iteration and `getCachedStringWidth(char)` (with `Math.max(..., 1)` floor for combining marks), accumulates a sliding-window of `visibleContentHeight = targetMaxHeight - 1` lines via `appendVisibleLine` + `visibleLines.shift()`, and returns `{text, hiddenLinesCount}` only when actual visual lines exceed the bound (early-return at `:144` `if (visualLineCount <= targetMaxHeight)`).
- `MIN_PENDING_PREVIEW_HEIGHT = 2` constant at `:84` is a sensible floor — prevents a one-row terminal from collapsing to zero visible content.
- `PendingTextPreview` component at `:153-170` renders `... first N streaming line(s) hidden ...` in `theme.text.secondary` with correct singular/plural pluralization (`hiddenLinesCount === 1 ? '' : 's'`).
- `PrefixedMarkdownMessage:208-242` and `ContinuationMarkdownMessage:253-280` both gate the slicer behind `isPending` — once streaming completes, the full content goes through `MarkdownDisplay` again (correctly preserving final markdown formatting). Both paths share the `markdownWidth = Math.max(1, contentWidth - prefixWidth)` floor and `effectiveTextColor = textColor ?? theme.text.primary` defaulting.
- The integration test at `integration-tests/terminal-capture/streaming-clear-storm.ts:191-560` (276 lines) drives a fake OpenAI-compatible streaming server, streams 220 chunks at 70ms/chunk against an 88×26 PTY with synchronized-output disabled (so terminal capability hiding cannot mask the underlying Ink behavior), captures 55 frames + raw ANSI, and asserts: `clearTerminalPairCount == 0`, `finalDoneCount == 1`, `framesCaptured >= 40`. That's the right pass/fail signal for a flicker class.
- Shell-side fix: `shellExecutionService.ts` render callback now requires both a semantic viewport change *and* new renderable output before sending another live shell chunk — closes the resize-soft-wrap-as-fresh-output false positive.

## Verdict

**merge-as-is**

## Rationale

This is exactly the right shape of fix for a TUI render-pipeline bug:

1. **Bounded scope.** The doc's "explicit non-claims" section means the next person who hits a different flicker class won't waste hours assuming this PR closed it.
2. **Pending vs. final separation.** The slicer only fires while `isPending`; the final transcript renders through unchanged `MarkdownDisplay`. This avoids the regression class where someone fixes streaming flicker by losing markdown formatting.
3. **Visual-width-aware accounting.** Using `getCachedStringWidth` + `toCodePoints` is the only correct way to count visible rows for streams that contain CJK, emoji, or zero-width joiners. A naive `text.split('\n').length` would miscount.
4. **Tail-preserving sliding window.** `visibleLines.shift()` keeps the newest content (which is what users actually need during streaming) rather than truncating the tail and showing stale older content.
5. **Deterministic acceptance metrics.** ANSI-level `clearTerminalPairCount == 0` is an objective pass/fail signal — not a screenshot diff that can be argued about. The 40-frame minimum prevents "fewer frames captured because the storm shortened the run" gaming the success metric.

The 808-line size is justified by the 170-line design doc + 276-line capture harness — the actual production change is roughly 130 lines in `ConversationMessages.tsx` plus a few lines in `shellExecutionService.ts`. Worth merging.
