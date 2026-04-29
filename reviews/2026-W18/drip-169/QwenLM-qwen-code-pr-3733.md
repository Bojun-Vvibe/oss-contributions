# QwenLM/qwen-code#3733 — `feat(cli): support batch deletion of sessions in /delete`

- PR: https://github.com/QwenLM/qwen-code/pull/3733
- Head SHA: `881eebfe6570f4b47e338bf8d4023c3043d5ee90`
- Author: qqqys

## Summary of change

Adds multi-select to the existing `SessionPicker` component and
wires `/delete` through it. New props: `enableMultiSelect`,
`onConfirmMulti(sessionIds[])`, `disabledIds[]`. UI changes:
checkbox column (`[x] ` / `[ ] `) reserves 4 chars of prompt
width; Tab toggles the cursor item; Enter commits the checked set
(falls back to single-select when nothing checked); current
session is added to `disabledIds` so it can't be batch-deleted.
A new `useDeleteCommand` hook exposes both `handleDelete` (single)
and `handleDeleteMany` (batch). 142 lines of new test cases.

## Specific observations

- `packages/cli/src/ui/components/SessionPicker.tsx:96-110` — the
  `SessionListItemViewProps` extension is well-scoped: three new
  optional fields with explicit JSDoc. The `isChecked` `undefined`
  vs `false` distinction (line 96 comment "When defined, render a
  leading `[x]`/`[ ]` checkbox") is a clean way to keep the
  single-select code path visually identical to before.
- `SessionPicker.tsx:131-138` — `checkboxWidth = isChecked === undefined ? 0 : 4` and the `Math.max(1, maxPromptWidth -
  checkboxWidth)` truncation. Good — prevents the prompt column
  from collapsing to negative width on a narrow terminal. Worth
  considering: when `maxPromptWidth - 4 < 8` or so (very narrow
  TTY), the truncated prompt becomes unreadable; a fallback that
  hides the checkbox below some threshold would be friendlier.
- `SessionPicker.tsx:159-178` — checkbox color logic: when
  `isDisabled`, secondary; when `isChecked`, accent; else
  secondary. The cascade reads correctly but it's three nested
  ternaries — extracting to a named function (`checkboxColor(isDisabled, isChecked, isSelected)`) would make this
  easier to maintain when the next state (e.g. "pending delete")
  appears.
- `SessionPicker.tsx:194` — `{isDisabled && disabledHint ? ` ·
  ${disabledHint}` : ''}` appends the localised
  `t('current — cannot delete')` to the metadata row. Good i18n
  hook.
- `DialogManager.tsx:418-432` — the multi-select wiring sets
  `disabledIds={currentSessionId ? [currentSessionId] : undefined}`.
  Correct invariant — the user can't delete the conversation they're
  in. But there's no equivalent guard against deleting a session
  that another `qwen` instance has open. Cross-process protection
  isn't in scope for this PR but is worth a TODO comment near the
  `disabledIds` construction so the next person doesn't assume
  this is the full guard.
- `StandaloneSessionPicker.test.tsx:907-1041` (the "Multi-select"
  describe block): covers Tab toggle, Enter commits checked,
  Enter with nothing-checked falls back to single-select, and
  Tab on a `disabledIds` row is no-op. Solid coverage. Missing:
  a test for `disabledIds` containing a session that's *not*
  currently visible (off-screen due to scroll) — does Tab still
  no-op when the underlying list pages?
- The "Tab toggles" / "Space to toggle" footer hint is
  inconsistent: the JSDoc on the prop (line 60) says "Tab toggles
  a checkbox" but the rendered hint at lines 380-386 says "Space
  to toggle". Pick one and be consistent in both the prop docs
  and the visible affordance. (Looks like the actual key handler
  is in `useSessionPicker` and the footer is correct — so the
  prop JSDoc is the one to fix.)

## Verdict

`merge-after-nits` — feature is the right shape, multi-select is
properly opt-in via a flag so existing single-select callers are
unaffected, and the `disabledIds` design naturally accommodates
the "current session can't be deleted" invariant. The Tab-vs-Space
JSDoc/hint mismatch must be fixed before merge to avoid user
confusion; the rest are non-blocking polish.
