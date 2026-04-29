# sst/opencode#25016 — fix(tui): prevent question option submission on drag-to-select copy

- PR: https://github.com/sst/opencode/pull/25016
- HEAD: `a42d24472832d03b211c08b77bd0cb909e3b1888`
- Author: euxaristia
- Files changed: 1 (+18 / -5) — `packages/opencode/src/cli/cmd/tui/routes/session/question.tsx`

## Summary

Fixes a TUI UX bug where drag-to-select on a question-prompt option
silently submitted that option (because `onMouseUp` fired the selection
callback) instead of letting the app-root copy-on-select handler copy the
selected text to the clipboard. The fix follows the established pattern
from other clickable surfaces in the TUI: guard each `onMouseUp` with
`if (renderer.getSelection()?.getSelectedText()) return`. Imports
`useRenderer` and applies the guard at four `onMouseUp` sites — the tab
options, the Confirm tab, the option list rows, and the "other" option
row.

## Cited hunks

- `packages/opencode/src/cli/cmd/tui/routes/session/question.tsx:3` —
  imports `useRenderer` from `@opentui/solid`.
- `packages/opencode/src/cli/cmd/tui/routes/session/question.tsx:124` —
  `const renderer = useRenderer()` at the top of `QuestionPrompt`,
  colocated with `const dialog = useDialog()`.
- `packages/opencode/src/cli/cmd/tui/routes/session/question.tsx:283-286`
  — tab-option `onMouseUp` becomes `if (renderer.getSelection()?.getSelectedText()) return; selectTab(index())`.
- `packages/opencode/src/cli/cmd/tui/routes/session/question.tsx:311-314`
  — Confirm tab `onMouseUp` gets the same guard wrapping
  `selectTab(questions().length)`.
- `packages/opencode/src/cli/cmd/tui/routes/session/question.tsx:338-341`
  — option-row `onMouseUp` calling `selectOption()` gets the guard.
- `packages/opencode/src/cli/cmd/tui/routes/session/question.tsx:370-373`
  — "other" option row `onMouseUp` gets the guard.

## Risks

- Pure UX patch; no behavioural risk to the keyboard path (which never
  goes through `onMouseUp`) or to fast-clicks (which produce no selection
  so the guard short-circuits).
- The guard is duplicated four times verbatim. If a fifth clickable
  surface gets added without the guard, the bug recurs silently. Trivial
  refactor: lift to a `withSelectionGuard(handler)` wrapper or a
  `useSelectionGuardedClick` hook colocated with the renderer access.
- `getSelection()?.getSelectedText()` is checked for truthiness. An empty
  string from a degenerate selection (e.g. zero-width drag) would
  correctly fall through to `selectOption()`, which matches both the
  other guarded sites and user intent (an accidental click submits;
  a deliberate selection doesn't).

## Verdict

**merge-as-is**

## Recommendation

Land it. The fix is mechanically correct, follows the existing pattern,
and the bug it fixes (especially in plan mode where multi-choice prompts
are common) is genuinely annoying. Refactor to a shared helper can land as
a follow-up sweep across all `onMouseUp` sites in the TUI tree without
blocking this fix.
