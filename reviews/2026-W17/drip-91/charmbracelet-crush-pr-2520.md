---
pr: 2520
repo: charmbracelet/crush
sha: a40140096abdb9aac3b27d3c88f71a595a4c7b4e
verdict: merge-as-is
date: 2026-04-27
---

# charmbracelet/crush #2520 — fix(ui): prompt to quit on ctrl+d

- **Head SHA**: `a40140096abdb9aac3b27d3c88f71a595a4c7b4e`
- **URL**: https://github.com/charmbracelet/crush/pull/2520
- **Size**: small (87/4 across 3 files; ~half of the diff is tests)

## Summary
Replaces the old behavior of Ctrl+D on an empty editor (which exited the TUI immediately) with a prompt-to-quit dialog. Introduces a `shouldPromptQuitOnEOF()` predicate that gates the dialog on a precise set of conditions (focus on editor, in chat or landing state, no buffered input, no completion popup, no attachments).

## Specific findings
- `internal/ui/model/ui.go:1679-1681` — the new branch fires inside `handleKeyPressMsg` *before* the existing `Cancel` check at `:1683`. Order matters here: if the user holds Ctrl while typing a message and accidentally hits D, we don't want the cancel-while-busy path to swallow the keypress. The current order is correct because `shouldPromptQuitOnEOF` returns false when `m.textarea.Value() != ""`.
- `ui.go:2957-2972` — the predicate is exhaustive in a good way. Each clause fails closed:
  - `msg.String() != "ctrl+d"` — sole entry condition.
  - `m.focus != uiFocusEditor` — prevents the dialog from popping while the user is, say, scrolling the chat history.
  - `m.state != uiChat && m.state != uiLanding` — keeps Ctrl+D inert during onboarding / settings flows where it might mean something else to a sub-component.
  - `m.textarea.Value() != "" || m.completionsOpen` — preserves the existing readline-like Ctrl+D behavior (delete-char-forward) when there's buffered input.
  - `m.attachments == nil || len(m.attachments.List()) == 0` — refuses to discard pending attachments by accident. The `m.attachments == nil` guard is what keeps `newTestUI()` working before the test setup fully populates attachments (and is also defensive against early-init paths).
- `internal/ui/model/ui_keypress_test.go` (new file) — three table-style tests covering: empty editor → dialog opens; non-empty editor → no dialog; pending attachment → no dialog. Solid coverage of the conditions in the predicate. The completion-open and `state != uiChat` cases aren't tested but the predicate is short enough that visual review is sufficient.
- `internal/ui/model/layout_test.go:30-66` — `newTestUI()` is updated to wire `keyMap`, `dialog.NewOverlay()`, and `attachments.New(...)`. This is the right shape — the previous `newTestUI()` was a minimal stub that would have crashed the new tests on `m.dialog.ContainsDialog(...)`. The keymap + attachments setup is reused for any future predicate test.
- One small thing: the predicate function name `shouldPromptQuitOnEOF` says "EOF" but the actual trigger is "ctrl+d on empty editor", not POSIX EOF semantics. Minor naming nit, not worth a re-review cycle.

## Risk
Trivial. The change is a strict UX improvement (no more accidental exits when reaching for backspace), the predicate is conservative (defaults to old behavior for any unusual state), and the test surface is appropriate for the size.

## Verdict
**merge-as-is** — clean predicate, good test coverage, the bundled `newTestUI()` plumbing change is a necessary precondition for the tests to compile and is itself harmless.
