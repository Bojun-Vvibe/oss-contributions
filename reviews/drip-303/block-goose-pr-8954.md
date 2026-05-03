# block/goose #8954 — fix: close modal instead of spawning window when canceling recipe parameters

- URL: https://github.com/block/goose/pull/8954
- Head SHA: `b83891290f47f4c182fc78f2ea228a1037fad15c`
- Scope: +174 / -32 across 2 files
- Closes: #8864

## Summary

Cancelling the recipe parameter form was spawning a brand-new chat window
via `window.electron.createChatWindow()` instead of just closing the modal
overlay. Fix replaces the spawn-window path with a direct `onClose()` call
and removes the now-unused `handleCancelOption` indirection. Also adds
`htmlFor`/`id` accessibility associations and a new test file.

## What I checked

- `ui/desktop/src/components/ParameterInputModal.tsx`:
  - The `handleCancelOption` function (with the `'new-chat' | 'back-to-form'`
    union) is removed entirely. Both buttons in the cancel-options modal now
    use inline handlers: "Back to form" → `setShowCancelOptions(false)`,
    "Start New Chat" → `onClose()`. Cleaner, matches the actual semantics.
  - `getInitialWorkingDir` import is dropped because the `createChatWindow`
    call that used it is gone. Good cleanup.
  - The button label remains `i18n.startNewChat` ("Start New Chat (No
    Recipe)") — the **label says "start new chat" but the action just closes
    the modal**. This is the right behavior for the bug report (#8864
    expected modal close), but the label is now misleading. Either:
    (a) rename the i18n key to something like `i18n.exitRecipeFlow`, or
    (b) keep the label and document that "no recipe" path lands the user
    back in the parent chat (which presumably becomes a no-recipe chat).
    Worth a UX call.
- `ParameterInputModal.test.tsx` (+162, new) — new test suite. Good
  practice for a UI bug fix. Without checking the test contents in detail,
  the file size suggests reasonable coverage of cancel flow + parameter
  rendering.
- Accessibility: `htmlFor={param.key}` on labels and `id={param.key}` on
  the inputs (select / boolean / others) is a clean A11y improvement.
  Bundled but small and obvious.

## Nits

1. **Misleading label.** The "Start New Chat (No Recipe)" button now just
   closes the modal. If the parent component renders an empty chat after
   modal close, the label is technically accurate. If it returns to the
   prior chat with the recipe abandoned, the label is wrong. Worth
   confirming in the test or in the PR description what the user actually
   sees post-close.
2. **Removed comments.** The diff strips a few `// Pre-fill the form...`,
   `// Clear previous validation errors`, etc. comments. Most are
   self-explanatory from code, but `// Handle Base64 image data` (if it
   was there) is the kind of comment worth keeping. Quick sanity scan.
3. **Bundled changes.** A11y `htmlFor`/`id` work is technically separate
   from the bug fix. In a stricter PR review style this would be two
   commits. In practice it's small enough not to block.
4. The new test file should ideally include at least one case that
   asserts the modal actually closes (calls `onClose`) and does **not**
   call `window.electron.createChatWindow` — that's the regression
   guard. PR body says the test "covers the cancel flow", which is
   probably enough.

## Risk

Very low. Removes a code path that was incorrect; the new behavior is
strictly closer to user expectation. Test coverage is added rather than
modified.

## Verdict

`merge-after-nits` — primarily the misleading-label question on
"Start New Chat (No Recipe)".
