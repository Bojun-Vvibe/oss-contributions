# anomalyco/opencode PR #24285 — fix(tui): add copy action for native question prompts

- **URL:** https://github.com/anomalyco/opencode/pull/24285
- **Author:** @jeevan6996
- **State:** OPEN (target `dev`)
- **Head SHA:** `8f9a70342e84c4c40c8941b4c9e6423e45f6dec6`
- **File:** `packages/opencode/src/cli/cmd/tui/routes/session/question.tsx`

## Summary of change

Adds a `c` keybinding to the native question prompt that copies the
question text to the system clipboard, with toast feedback. Also
exposes the action in the prompt's footer hint row with an
`onMouseUp` handler.

## Findings against the diff

- **L12–13:** new imports — `Clipboard` from `../../util/clipboard`
  and `useToast` from `../../ui/toast`. Both already exist in the
  project (used in adjacent components), so no new deps.
- **L126–134 `copyQuestion()`:**
  ```ts
  const text = question()?.question?.trim()
  if (!text) {
    toast.show({ message: "No question text to copy", variant: "error" })
    return
  }
  Clipboard.copy(text)
    .then(() => toast.show({ message: "Question copied to clipboard", variant: "success" }))
    .catch(() => toast.show({ message: "Failed to copy question", variant: "error" }))
  ```
  Clean. Empty/whitespace-only question short-circuits with a
  toast. No `await` so the keybind handler returns immediately —
  correct since the toast handles user feedback async.
- **L229–234 keybind block:** correctly guards with
  `!evt.ctrl && !evt.meta && !evt.shift` so it only fires on a
  bare `c`. Uses `evt.preventDefault()` + `return` to avoid falling
  through into the option-picker numeric-shortcut block below.
- **L482–486 footer hint:** wrapped in `<Show when={!confirm()}>`
  so the hint disappears in the confirm sub-state — matches the
  pattern of the surrounding `esc dismiss` etc.
- **Conflict risk:** `c` is a single, common letter. If the question
  prompt ever supports text-input mode (custom answer typing),
  pressing `c` would now trigger copy instead of typing. The
  current code path is gated by the `else` branch of an outer
  `if (custom() ...)` so we only see this in the *picker* phase —
  safe today.

## Verdict

**merge-as-is**

Small, well-scoped UX fix. Toast copy is consistent with the rest
of the TUI. Only a stylistic nit (not a blocker): could use
`async/await` instead of `.then/.catch` for symmetry with other
clipboard sites in the codebase, but the chained form is fine.
