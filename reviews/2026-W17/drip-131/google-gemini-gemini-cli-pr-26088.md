# Review — google-gemini/gemini-cli #26088

- **Title**: fix(cli): add F10 fallback for approval mode cycling
- **Author**: Gitanaskhan26 (Anas Khalid)
- **Head**: `1713aca82caa26552bbedc4d8ff0552d07c651c8`
- **Verdict**: merge-after-nits

## Summary

Three-line fix at `packages/cli/src/ui/key/keyBindings.ts:399-432` adding `F10` as an additional `KeyBinding` alongside `Shift+Tab` for three Commands: `CYCLE_APPROVAL_MODE`, `UNFOCUS_SHELL_INPUT`, and `UNFOCUS_BACKGROUND_SHELL`. Closes #22738. Motivation: `Shift+Tab` escape sequences are mis-parsed by some Windows/WezTerm setups, so users currently can't cycle approval modes.

## Findings

- **Right shape** — `KeyBinding[]` arrays already exist for `Command.RESTART_APP` (`'r'`, `'shift+r'`) so adding multiple bindings per Command is the established pattern; no new dispatch logic needed. Pure data change, zero behavior risk for the `Shift+Tab` path.
- **Choice of F10 is well-motivated** — F-keys are reported as `\e[21~` (DECKPAM-style) in xterm-compatible terminals and are historically reliable across Windows console hosts where modifier-key sequences (CSI `Z` for `Shift+Tab`) get mangled. F10 specifically is rarely bound by modern terminal emulators (F11=fullscreen toggle is the more common collision). Good pick.
- **Symmetry across all three commands is correct** — `CYCLE_APPROVAL_MODE`, `UNFOCUS_SHELL_INPUT`, and `UNFOCUS_BACKGROUND_SHELL` all share `Shift+Tab` as their primary binding; adding F10 to *only* `CYCLE_APPROVAL_MODE` would create the surprising state where Windows users could cycle approval but couldn't escape shell focus with the same fallback. Author correctly identified all three sites.
- **Potential collision risk not investigated**: F10 in many terminals (gnome-terminal, KDE Konsole, default macOS Terminal.app menu bar mnemonic activation) opens a menu — verify on the unchecked MacOS/Linux platforms. PR's testing matrix shows only Windows checked. Land an explicit smoke note in the PR body confirming F10 doesn't break those platforms or document the limitation.
- **No test added** — `keyBindings.ts` tests would just be a structural assertion that `defaultKeyBindingConfig.get(Command.CYCLE_APPROVAL_MODE)` includes both bindings, which is low value but pins the correct invariant: a future "let's clean up the array shape" refactor would silently drop F10. One-cell test is appropriate.
- **No new menu/help-text update** — if there's a `/help` or keyboard-shortcuts cheat-sheet that documents `Shift+Tab`, it should now mention "or F10". Search for `'shift+tab'` string-literal usages in user-facing docs and update.
- **Interaction with `TOGGLE_COPY_MODE` at `:399`** which uses `f9` — F9 and F10 being adjacent on the keyboard is generally fine but if a user holds F-keys for repeated cycling, F9→F10 typo cost is now "toggle copy mode" → "cycle approval". Low probability, mention in changelog.

## Recommendation

Merge after (a) confirming F10 doesn't conflict with terminal menu activation on macOS/Linux (or documenting the platform-specific caveat), (b) adding a one-cell test asserting all three Commands include the F10 binding, and (c) updating any user-facing keyboard-shortcuts documentation. The fix is correct, the choice of F10 is well-motivated, and the symmetry across the three commands is the right invariant.