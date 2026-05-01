# block/goose #8954 — fix: close modal instead of spawning window when canceling recipe parameters

- **Repo:** block/goose
- **PR:** https://github.com/block/goose/pull/8954
- **HEAD SHA:** `e9ae223c25bb07893840cfdffe7b775dbfcf13fc`
- **Author:** enilsen16
- **Verdict:** `merge-after-nits`

## What the diff does

In `ui/desktop/src/components/ParameterInputModal.tsx`:

1. **Removed** the `handleCancelOption` helper at `:127-141` plus
   the `getInitialWorkingDir` import at `:4`. Previously, when a
   user opened the recipe-parameters modal, then clicked Cancel,
   then chose "New chat" from the cancel-options sub-screen, the
   handler called `window.electron.createChatWindow({dir:
   workingDir})` and `window.electron.hideWindow()` — i.e.
   spawned a *new* electron window for an empty chat instead of
   just dismissing the modal.

2. **Replaced** the two cancel-option button click handlers at
   `:137,145`:
   - "Back to form" button now calls
     `() => setShowCancelOptions(false)` directly (was
     `handleCancelOption('back-to-form')`).
   - "New chat" button now calls `onClose` directly (was
     `handleCancelOption('new-chat')`).

3. **Accessibility wins** at `:166,193,208,220` — adds `htmlFor` on
   the `<label>` and matching `id` on each `<input>` / `<select>`
   so screen readers can properly associate parameters with their
   labels.

4. **New test file** `ui/desktop/src/components/__tests__/ParameterInputModal.test.tsx`
   (+168 lines) — covers rendering, required-field validation,
   form submission with parameter values, and the new cancel
   behavior.

## Why the change is right

The previous "New chat" behavior was a UX surprise pattern: a user
who decides "I don't want to fill out these parameters" expects
the modal to close and return them to whatever they were doing.
Spawning a brand-new electron window for an empty chat is a
heavyweight side-effect that:

1. Loses focus/state of the user's current session.
2. Interacts oddly with multi-window users (existing windows stay
   open, new empty one piles on).
3. Required `getInitialWorkingDir()` to be available, which couples
   modal dismissal to filesystem state.
4. The label "New chat" suggests *navigation*, not *spawn-and-
   hide* — the verb mismatch was itself a bug.

Replacing both arms with the natural primitives (`onClose` for
dismissal, `setShowCancelOptions(false)` for back-navigation)
removes the surprise, deletes the dead `getInitialWorkingDir`
import, and reduces the surface area of `window.electron.*` calls
the modal needs to know about — all of which are net-positive
architectural moves alongside the UX fix.

The label/input pairing changes (`htmlFor` + `id`) are honest
collateral a11y wins. They share the parameter `key` as the DOM
id, which is correct because parameter keys are unique within a
recipe (the existing form-validation logic at `:124` keys on
`param.key` for the same reason).

## Nits (non-blocking)

1. **"New chat" button label is now misleading** at `:148`. After
   this change, clicking "New chat" just calls `onClose` — it
   doesn't actually start a new chat. It dismisses the modal back
   to whatever underlying view the user came from. Consider
   relabeling to "Cancel" / "Close" / "Discard" so the button
   text matches what it now does. Otherwise users who click "New
   chat" will be confused when the modal closes and they're back
   on the same chat as before.

2. **Test imports `useState`'s setShowCancelOptions(false)
   directly via `() =>`** — but the test file at +168 lines is
   long enough that the diff doesn't show whether it covers the
   "click 'New chat' calls onClose" arm specifically. Should
   include:
   - `it("calls onClose when 'New chat' is clicked", ...)` —
     pin the exact behavior change of this PR.
   - `it("does NOT spawn an electron window when 'New chat' is clicked", ...)` —
     assert `window.electron.createChatWindow` is never called
     (regression-pin against someone restoring the old behavior).
   The test file as shown covers rendering / validation /
   submission, but the cancel-behavior tests need to be the
   load-bearing assertion since that's the bug being fixed.

3. **Parameter-key DOM id collision risk.** Recipes can have a
   parameter named `submit`, `cancel`, `id`, or any other DOM-
   reserved-ish word. `id={param.key}` puts user-controllable
   strings into DOM ids without sanitization. For most cases this
   is harmless, but if any other code does
   `document.getElementById("submit")` it would hit the input
   instead of an intended submit button. Worth either prefixing
   (`id={`param-${param.key}`}`) or documenting that recipe
   authors mustn't use these names.

4. **`AppearanceSettings.tsx` grid→flex-wrap** is mentioned in
   drip-251 #8952's review — same author, same time window,
   different concern. If similar drive-by-style changes appear
   in this PR they should split out; the diff above looks
   focused so this is just a "watch out for scope creep" note.

5. **Removal of the `getInitialWorkingDir` dependency** is good
   but worth one line in the PR description noting that any
   downstream component that relied on the modal's "New chat"
   side-effect to also set their working dir is now on its own.
   If no downstream component relied on it (likely, given how
   surprising the original behavior was), no action needed.

## Verdict rationale

Right diagnosis (modal dismissal shouldn't spawn a window),
clean removal (delete the helper, route the buttons to natural
primitives), honest collateral a11y improvements, and a new
test file. The button-label mismatch is the one user-facing
loose end; the DOM-id sanitization is a defensive nit; the
test-coverage assertion that the new cancel arm calls `onClose`
should be load-bearing.

`merge-after-nits`
