# charmbracelet/crush #2520 — fix(ui): prompt to quit on ctrl+d

- **Head SHA:** `a40140096abdb9aac3b27d3c88f71a595a4c7b4e`
- **Base:** `main`
- **Author:** officialasishkumar
- **Size:** +87 / −4 across 3 files (1 layout test rig, 1 ui.go, 1 new keypress test file)
- **Verdict:** `merge-as-is`

## Summary

Closes #2428: `Ctrl+D` on an empty editor with no pending attachments now
opens the existing quit-confirmation dialog, treating `Ctrl+D` as a REPL-style
EOF. Previously `Ctrl+D` was ignored unless the user typed `quit` or used
`Ctrl+C`. Behavior is gated to four conditions (right key, editor focused,
chat-or-landing state, empty textarea + closed completions + no attachments).

## What's right

- **Predicate is precisely scoped.** `shouldPromptQuitOnEOF` at
  `internal/ui/model/ui.go:2957-2970` enforces five independent gates that
  must all hold:
  1. `msg.String() == "ctrl+d"` (line 2958-2960)
  2. `m.focus == uiFocusEditor` (line 2961-2963) — won't fire when focus is
     on the chat list, completions, dialog, etc.
  3. `m.state == uiChat || m.state == uiLanding` (line 2964-2966) — won't
     fire during quit dialog, settings, init, etc.
  4. `m.textarea.Value() == "" && !m.completionsOpen` (line 2967-2969) —
     respects the typed-input invariant the bug report calls out
  5. `m.attachments == nil || len(m.attachments.List()) == 0` (line 2970)
     — won't fire when an attachment is staged for send

  This is exactly the right shape for "treat Ctrl+D like EOF only when the
  prompt looks like a fresh REPL line."

- **Re-uses the existing quit dialog** at `ui.go:1679-1681` (the existing
  `m.openQuitDialog()` helper). Doesn't introduce a new exit path that
  could diverge from the `quit` slash-command and `Ctrl+C` paths in the
  future.

- **Compact-mode `Ctrl+D` (toggle details) is not regressed.** The PR body
  explicitly notes "Existing compact-mode `ctrl+d` details behavior is
  unchanged outside the empty-editor EOF case." Verified against the
  predicate: when compact-mode is showing details, `m.focus` is not
  `uiFocusEditor`, so gate 2 trips and `shouldPromptQuitOnEOF` returns
  false — the existing details handler downstream still sees the keypress.

- **Three-test coverage maps 1:1 to the predicate's branches.**
  `ui_keypress_test.go` (new file, 45 lines, parallel-safe via
  `t.Parallel()`):
  - `TestHandleKeyPressMsgCtrlDOpensQuitDialogWhenEditorIsEmpty` — happy
    path, asserts `ui.dialog.ContainsDialog(dialog.QuitID)`.
  - `TestHandleKeyPressMsgCtrlDDoesNotOpenQuitDialogWhenEditorHasInput` —
    pins gate 4 (typed-input) by setting `ui.textarea.SetValue("hello")`.
  - `TestHandleKeyPressMsgCtrlDDoesNotOpenQuitDialogWhenAttachmentsExist`
    — pins gate 5 (pending attachments) by calling
    `ui.attachments.Update(message.Attachment{…})`.

  The two negative tests use `require.False(...)` against the same
  `ContainsDialog(QuitID)` check the positive test uses, so a future
  refactor that swaps the dialog ID will fail all three together rather
  than diverging.

- **`newTestUI()` at `layout_test.go:32-69`** is updated to provide the
  four pieces the new predicate touches that the prior fixture didn't
  carry: `keyMap` (line 33, used for `m.keyMap.Editor.*` references in
  the attachments keymap, line 56-60), `dialog` (line 38, the overlay
  the new code asserts the dialog ID against), and `attachments` (line
  41-58, the manager whose `.List()` length is checked at gate 5). This
  is a real test-rig debt repayment — without it the new tests can't
  even compile.

- **`attachments` constructor wiring is faithful to production.** Lines
  42-58 build `attachments.New(attachments.NewRenderer(…),
  attachments.Keymap{…})` with the same renderer styles
  (`com.Styles.Attachments.{Normal,Deleting,Image,Text}`) and keymap
  bindings (`AttachmentDeleteMode`, `DeleteAllAttachments`, `Escape`)
  that the production wiring uses. So the test rig will exhibit the
  same attachment-state behavior as the running TUI under
  `Update`/`List` mutations.

## Concerns / nits

None blocking. Optional polish:

1. **`shouldPromptQuitOnEOF` could short-circuit on `msg.Code` first.**
   `msg.String()` is the user-facing rendering (e.g. allocates the
   formatted `"ctrl+d"` string on every keypress). Comparing
   `msg.Code == 'd' && msg.Mod == tea.ModCtrl` first would avoid the
   string conversion on the 99.9% of keypresses that aren't Ctrl+D. The
   tests at lines 14-17 and 25-28 already use the
   `tea.KeyPressMsg{Code: 'd', Mod: tea.ModCtrl}` shape, so they'd keep
   passing under either comparison style.

2. **Test for the focus-gated case is missing.** Gates 1, 4, and 5 are
   pinned by tests; gates 2 (`uiFocusEditor`) and 3 (`uiChat ||
   uiLanding`) are implicit (the test rig ships with `state: uiChat,
   focus: uiFocusEditor` per `layout_test.go:64-65`). A fourth test
   that flips `ui.focus = uiFocusChatList` (or similar) and asserts the
   dialog is NOT opened would close the loop on gate 2. Same for an
   `uiSettings` state test. Optional — the predicate is small enough to
   eyeball.

3. **`m.completionsOpen` is conjuncted with `m.textarea.Value() == ""`
   on the same line (2968).** If a future change adds another
   "completion-like overlay" (e.g. a slash-command picker), the gate
   would need to be extended. A `m.hasOpenOverlay()` helper would
   centralize this. Pre-existing pattern, not introduced here.

## Verdict rationale

Tiny, focused fix for a well-defined REPL UX gap. Predicate is correctly
scoped (5 explicit gates), three negative-and-positive tests cover the
three branches that aren't pre-conditioned by the test rig, and the
existing quit dialog is reused so no exit-path divergence. No reason to
withhold; `merge-as-is`.
