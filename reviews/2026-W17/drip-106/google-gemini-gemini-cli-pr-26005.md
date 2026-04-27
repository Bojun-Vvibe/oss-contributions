# google-gemini/gemini-cli PR #26005 — Fix infinite dialog loop and add ESC support in /skills link

- **Link**: https://github.com/google-gemini/gemini-cli/pull/26005
- **Head SHA**: `32f1466e5b13f57f8e0d1cf055a03509de5a2ac9`
- **Author**: JayMeDotDot
- **Size**: +47 / -7 across 5 files
- **Verdict**: `merge-after-nits`

## Files changed
- `packages/cli/src/ui/commands/skillsCommand.ts:115-126` — wraps the previously-bound `setConfirmationRequest.bind(context.ui)` invocation with an explicit closure that, after the user's `onConfirm(confirmed)` runs, calls `context.ui.setConfirmationRequest(null)` to dismiss the dialog. The original `.bind()` form passed the request through unchanged, never clearing the slot, so the dialog component stayed mounted indefinitely.
- `packages/cli/src/ui/commands/types.ts:89-92` — widens `setConfirmationRequest` signature from `(value: ConfirmationRequest) => void` to `(value: ConfirmationRequest | null) => void`. This is the type-level admission that "no active request" is a valid state — previously the type forbade clearing the slot at all.
- `packages/cli/src/ui/components/ConsentPrompt.tsx:23-37` — adds a `useKeypress` hook that matches `Command.ESCAPE` via `keyMatchers[Command.ESCAPE](key)` and calls `onConfirm(false)` on match. Returns `true` from the handler to mark the key as consumed.
- `packages/cli/src/ui/components/ConsentPrompt.test.tsx` — switches imports from the bare `render` helper to `renderWithProviders as render` (the previous helper didn't expose the keypress provider), and adds a 19-line `it('calls onConfirm with false when ESC is pressed', ...)` that writes the `\x1b` escape byte to stdin via `act(async () => { stdin.write('\x1b'); })` and asserts `expect(onConfirm).toHaveBeenCalledWith(false)`.
- `packages/cli/src/ui/noninteractive/nonInteractiveUi.ts:39-43` — non-interactive UI's no-op stub updated to the new `ConfirmationRequest | null` signature.

## Analysis

This is a two-fix-in-one PR but the two fixes are tightly coupled and share a common root cause: the `ConsentPrompt`/`ConfirmationRequest` machinery was designed assuming a single-shot dialog whose lifecycle was managed externally, but the actual usage in `skillsCommand.ts` never had an external manager calling `setConfirmationRequest(null)` after the dialog resolved. So:

1. **Bug 1 (infinite dialog loop)**: User runs `gemini skills link /path/to/repo`. `linkAction` calls `requestConsentInteractive(consentString, setConfirmationRequest.bind(context.ui))`. The dialog renders. User clicks Yes/No. The user's `onConfirm` callback fires inside `requestConsentInteractive`, returning the consent decision. **But nobody ever calls `setConfirmationRequest(null)`** — the request slot stays populated, the dialog component stays mounted, the user is stuck.

2. **Bug 2 (ESC docs lie)**: The `DialogFooter` (rendered inside `ConsentPrompt`) advertises ESC as a way to dismiss, but `ConsentPrompt` didn't actually have an ESC keypress handler. The footer was probably a copy-paste from another dialog that did handle ESC.

The fix shape is correct on both:

**For bug 1**, the right architectural change is to widen the type to `ConfirmationRequest | null` so "the dialog is dismissed" becomes a representable state, then have the request-issuing call site (`skillsCommand.ts:linkAction`) wrap the user's `onConfirm` with a thunk that clears the slot before delegating. Doing this at the call site rather than inside `ConsentPrompt` itself is the right layering: `ConsentPrompt` doesn't know about the `CommandContext`'s state machine, only the caller does. The 5-files-changed cost is mostly the type-update fanout: every consumer of the union now has to handle the `null` case (`nonInteractiveUi.ts` no-op stub).

**For bug 2**, hooking `useKeypress` inside `ConsentPrompt` with `keyMatchers[Command.ESCAPE](key)` and `onConfirm(false)` is the right shape — ESC = decline = same as "No" radio selection. Returning `true` from the handler marks the key as consumed so it doesn't bubble up to outer key handlers (e.g., a global `Ctrl+C` handler that might otherwise see it).

The test for bug 2 is the right minimum: write `\x1b` to stdin (the actual ESC byte), `await waitUntilReady()`, assert `onConfirm` called with `false`. The switch from `render` to `renderWithProviders` is necessary because `useKeyMatchers`/`useKeypress` require provider context — the original test would have thrown a "hook outside provider" error otherwise.

## Nits to address before merge

1. **No test for bug 1 (the dialog dismissal)**. The PR has tests for ESC but not for the actual headline bug — that after `onConfirm(true)` or `onConfirm(false)` resolves, `setConfirmationRequest(null)` is called and the dialog unmounts. A test that mocks `context.ui.setConfirmationRequest`, runs `linkAction`, simulates the consent decision, and asserts `setConfirmationRequest` was called twice (once with the request, once with `null`) would lock the contract. Without this test, a future cleanup PR could re-introduce the `.bind()`-style direct passthrough and re-break the dismissal.
2. **The `null` case in `nonInteractiveUi.ts`** is a no-op (correct — non-interactive has no dialog to dismiss), but consider asserting via test that calling `setConfirmationRequest(null)` doesn't throw in the no-op stub. One-line addition.
3. **The new closure shape in `skillsCommand.ts:118-125`** is correct but loses the `.bind()`-style readability. A one-line comment explaining *why* the closure is needed (`// Explicitly clear the request slot after user resolves; otherwise the dialog stays mounted (#issue-num)`) would prevent the next "let me clean up this awkward closure" PR.
4. **Other commands that issue confirmation requests** (a quick `rg 'requestConsentInteractive\|setConfirmationRequest' packages/cli/src/ui/commands/`) likely have the same lifecycle bug — the type widening to `| null` enables the fix everywhere but only `skillsCommand.ts` is updated. A follow-up audit issue would be appropriate; not necessarily blocking this PR.
5. **ESC during a pending async operation** (e.g., user hits ESC while the consent is being computed but before the dialog renders) — does `onConfirm(false)` fire on a still-mounting component? The `useKeypress(..., { isActive: true })` is unconditionally active, so probably yes. Verify there's no race where ESC gets consumed before the dialog is fully interactive.

## What I learned

The "type system forbade the cleanup" pattern is worth naming: when `setConfirmationRequest: (value: ConfirmationRequest) => void` doesn't accept `null`, the only way to "dismiss" the dialog at the type level is to set a *new* request — which is exactly the wrong escape hatch and leaves no representation for "no dialog active." Widening the type to `| null` (or `| undefined`, depending on house style) is the prerequisite for fixing the lifecycle bug, and doing the type widening + behavior fix in the same PR is the right grouping. The `ConsentPrompt` ESC bug, while logically separable, shares enough of the same surface (consent flow, dialog dismissal) that bundling makes sense — and the dismissal behavior of "ESC = onConfirm(false)" depends on the type widening to actually cause the dialog to disappear.
