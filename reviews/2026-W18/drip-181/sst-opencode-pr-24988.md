# sst/opencode#24988 — fix(tui): preserve text selection on question prompt options

- **Repo**: sst/opencode
- **PR**: [#24988](https://github.com/sst/opencode/pull/24988)
- **Head SHA**: `230f82a98abe987ea84c3e232e9d35064d7bb55b`
- **Author**: nikhil-patel-whatnot
- **Diff stats**: +18 / −5 (1 file)

## What it does

When the user is mid-drag selecting text inside a `QuestionPrompt` (e.g.
copying out a multiple-choice option label), the `onMouseUp` handlers on the
tab/option/"Confirm"/"other" boxes were unconditionally firing
`selectTab(...)` / `selectOption()`. That instantly committed the prompt and
the in-progress text selection was lost. Fix wires `useRenderer()` and
short-circuits each `onMouseUp` when `renderer.getSelection()?.getSelectedText()`
returns a non-empty string.

## Code observations

- `packages/opencode/src/cli/cmd/tui/routes/session/question.tsx:18` — adds
  `useRenderer` to the existing `@opentui/solid` import; correct, no new
  dependency surface.
- `packages/opencode/src/cli/cmd/tui/routes/session/question.tsx:280-285` —
  tab `onMouseUp` now bails when `renderer.getSelection()?.getSelectedText()`
  is truthy. The optional-chain on `getSelection()` is the right shape since
  the renderer may return `null`/`undefined` before any selection has ever
  been started.
- `:308-313`, `:335-341`, `:367-373` — same guard applied to the Confirm
  button, the per-option box, and the "other" option box. All four call sites
  use the identical predicate, which is good for consistency but invites
  extraction (see follow-ups).
- The guard is **only** on `onMouseUp`. `onMouseDown` (`:336`, `:368`) still
  calls `moveTo(i())` mid-drag, which means the highlighted/active option
  visually tracks the drag origin. That's acceptable UX — the *commit* is
  what's gated — but worth a brief PR-body line so the next reviewer doesn't
  read it as inconsistency.
- No regression test. The behavior is interactive-only and `@opentui` doesn't
  appear to ship a renderer mock for selection state, so a unit test would
  need a fake renderer.

## Verdict: `merge-after-nits`

The fix is correct, minimal, and matches a real UX bug. Nits:

1. Extract the predicate to a local `const isSelecting = () =>
   !!renderer.getSelection()?.getSelectedText()` near the top of the
   component so the four call sites read as `if (isSelecting()) return` and a
   future fifth interactive box can't forget the guard.
2. Add a one-line code comment at the first call site
   (`:280-285`) explaining the behavior, since "if there's a selection, don't
   commit" is a non-obvious mouseup contract that future refactors will
   delete as dead code.
3. Audit `prompt.tsx` and any other interactive TUI route for the same
   pattern — input selection is a cross-cutting concern, and if `Question`
   had the bug the slash-command palette and session list almost certainly
   do too.
4. If `@opentui` ever exposes a `RendererProvider` test harness, add a
   regression test stubbing `getSelection()` to return a fake `Selection`
   and asserting `selectOption` is not called.

## Follow-ups

- File a meta-issue tracking "TUI mouse handlers must respect active text
  selection" with a checklist of all interactive boxes.
- Consider whether `@opentui/solid` should expose a `useTextSelection()`
  hook so consumers don't have to thread the renderer for this single check.
