# google-gemini/gemini-cli#26202 — fix(cli): refine platform-specific undo/redo and smart bubbling for WSL

- **PR:** https://github.com/google-gemini/gemini-cli/pull/26202
- **Author:** @cocosheng-g
- **Head SHA:** `2da7598f` (full: `2da7598f1085926464f34313694ae705f1f9d567`)
- **Size:** +326/-32 across 8 files
- **Fixes:** #23648

## What this does

Three coupled changes around input-buffer undo/redo:

1. **Platform-specific keybindings reshuffle**
   (`packages/cli/src/ui/key/keyBindings.ts` +38/-9). On Linux/WSL the
   primary undo is now `Alt+Z` (and redo `Alt+Shift+Z`) because Windows 11
   intercepts `Win+Z` for Snap Layouts on the WSL window. On native Windows
   the primary stays `Ctrl+Z` / `Ctrl+Y`. macOS keeps `Cmd+Z` /
   `Shift+Cmd+Z`. The platform branching is asserted in the new
   `'should have platform-specific UNDO bindings'` test
   (`keyBindings.test.ts:111-138`).

2. **Smart bubbling for `Ctrl+Z`** (`packages/cli/src/ui/components/shared/text-buffer.ts`
   +10). The text buffer's `handleInput` now consults `undoStack.length` /
   `redoStack.length` before consuming the keypress:

   ```
   if (keyMatchers[Command.UNDO](key)) {
     if (undoStack.length === 0) {
       return false;
     }
     undo();
     return true;
   }
   ```

   That means when the input buffer is empty, `Ctrl+Z` is *not* consumed
   and bubbles up — preserving the conventional terminal-suspend
   (`SIGTSTP`) behavior on WSL. This is exactly the kind of contract that
   needs the corresponding test, which the PR adds:
   `'should only handle Undo if there is something to undo'`
   (`text-buffer.test.ts:1803-1862`) walks initial-state → insert → undo
   → "nothing left" and asserts `handled === false` at the right
   moments. The redo sibling test does the same at `1864-1932`.

3. **Docs + generator** (`docs/reference/commands.md`,
   `docs/reference/keyboard-shortcuts.md`,
   `scripts/generate-keybindings-doc.ts` +48/-5). The doc text is
   updated to call out the platform-specific mappings, and the generator
   gets new logic to render the `Ctrl+Z`/`Alt+Z`/`Cmd/Win+Z` matrix.

## What I like

- **The smart-bubbling pattern is the right primitive.** Returning
  `false` from `handleInput` so the platform suspend handler still gets
  the keystroke is the only correct behavior for terminal apps. Without
  it, WSL users with empty buffers couldn't suspend, which is the
  reported bug.
- **The deps array on `useTextBuffer`'s memoized `handleInput` was
  updated to include `undoStack.length, redoStack.length`** at
  `text-buffer.ts:3496-3498`. Without this, the smart-bubble check would
  read a stale closure value and the bug would reappear non-deterministically
  on re-renders. Easy to miss; nice catch.
- **The keyboard-shortcuts doc** (`docs/reference/keyboard-shortcuts.md`)
  is updated to render `Ctrl+Z` ahead of `Alt+Z` ahead of `Cmd/Win+Z`,
  which matches the actual platform precedence rather than the prior
  arbitrary ordering. This is meaningful — readers will assume the
  leftmost binding is the canonical one.

## Concerns

1. **`Ctrl+Z` is now ambiguous on WSL.** With smart bubbling,
   `Ctrl+Z`-when-buffer-has-text calls `undo()`, but `Ctrl+Z`-when-buffer-is-empty
   suspends the process. That's *probably* what users want, but it's also
   a footgun: a user typing fast may hit `Ctrl+Z` thinking they're going
   to undo their last keystroke and instead hit it after a `Ctrl+U`
   cleared their line, suspending their session mid-thought. Worth
   documenting this exact behavior in `commands.md` so it's not a
   surprise. The current docs change just lists the keys; it doesn't
   explain the bubble-on-empty contract.

2. **Test coverage for the non-undo/non-redo-related smart bubbling.**
   The PR only tests undo/redo. But adding the `undoStack.length` /
   `redoStack.length` deps to the `useMemo` is a change that affects
   *every* keystroke (the memoized handler now refreshes more often).
   Worth a perf check or at least a comment explaining that the recompute
   is bounded (`undoStack.length` is monotonic-ish per session).

3. **Unrelated "smart bubbling" applies only to UNDO/REDO.** The PR title
   says "smart bubbling for WSL" generally, but in code it's specifically
   undo/redo. Other potentially-conflicting bindings (`Ctrl+C`,
   `Ctrl+D`) are unaffected. That's fine, but the PR description
   over-promises. Adjust the wording or scope-narrow the title to
   "smart bubbling for undo/redo".

4. **macOS `Cmd+Z` test sequence.** The test at `text-buffer.test.ts:1810-1838`
   constructs the platform-specific key with hardcoded escape sequences
   like `'\u001b[122;D'` for darwin. Those are `xterm`-style modifyOtherKeys
   sequences. If the test environment doesn't run with that exact terminal
   profile, the macOS branch may exercise a different path than real
   users hit. Worth confirming with a Mac reviewer.

## Verdict

**`merge-after-nits`** — the fix is correct, the test additions are
substantial and exercise the actual contract (not just keybinding
existence), and the deps-array fix shows the author understood the
re-render trap. Just want the doc to call out the bubble-on-empty
behavior so users aren't surprised by `Ctrl+Z` suspending their session.

## Nits

- Document the `Ctrl+Z` bubble-on-empty contract explicitly in
  `commands.md`.
- Narrow the PR title to "...undo/redo" since smart bubbling only
  applies to those two commands.
- Confirm the macOS keypress fixtures match real-world terminal
  emulator output.
