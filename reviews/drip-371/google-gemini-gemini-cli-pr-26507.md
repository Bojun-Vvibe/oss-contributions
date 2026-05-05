# google-gemini/gemini-cli PR #26507 — fix(cli): prevent settings dialog border clipping using maxHeight

- URL: https://github.com/google-gemini/gemini-cli/pull/26507
- Head SHA: `4bbd28e4b7e1c782d36c514e5fbaada202f2274d`
- Author: jackwotherspoon (Jack Wotherspoon)
- Files: `packages/cli/src/ui/components/shared/BaseSettingsDialog.tsx` (1-line fix) + `SettingsDialog.test.tsx` regression test + 2 snapshots (+112/-1)

## Assessment

This is exactly the right shape for an Ink/flexbox layout fix: one-line behavior change + targeted regression test + visual snapshot. The fix at `BaseSettingsDialog.tsx:428` swaps `height="100%"` for `maxHeight={availableHeight}`. The diagnosis in the PR description is correct — `height="100%"` in Ink's yoga-based flex layout forces the box to consume its full computed parent height regardless of content, and when the parent has constrained height, sibling elements (the "Ctrl+C to exit" warning) get pushed into overflow and the bottom border gets clipped. `maxHeight={availableHeight}` lets the box shrink-to-fit when the content is shorter than the available terminal height, while still capping growth to terminal bounds when content is longer.

The new test at `SettingsDialog.test.tsx:329-356` is well-constructed: it sets `availableTerminalHeight: 15` (deliberately small to force the bug condition), then asserts both (a) `lines.length <= 15` proving the constraint is honored, and (b) the last line matches `/[─╰╯]/` proving the bottom border is rendered (these are the round-border code points used by the dialog). The two-pronged assertion catches both directions of regression — if a future change makes the dialog overflow OR if a change makes it stop rendering the border at all, the test fails. Pairing it with `toMatchSvgSnapshot()` adds visual diff coverage.

The snapshot at `SettingsDialog-...-bottom-border-correctly-when-height-is-constrained.snap.svg` shows the rendered dialog at 15-row height, with the bottom border `╰─...─╯` clearly present at `y="240"`. Cross-checking against the text snapshot in `SettingsDialog.test.tsx.snap` confirms the same structural output. One minor nit: the constrained-height test uses 15 rows which is the exact constraint, so the test is asserting "renders within budget" but not "behaves correctly when content exceeds budget" — a second test case at e.g. height 8 (where content genuinely can't fit) would lock down the overflow handling too. Optional follow-up.

The fix is non-invasive (one prop swap), well-tested, and the snapshot diffs are reviewable. Risk of regression elsewhere is low: `maxHeight` is more permissive than `height="100%"` so existing normal-terminal cases still render identically.

## Verdict

`merge-as-is`
