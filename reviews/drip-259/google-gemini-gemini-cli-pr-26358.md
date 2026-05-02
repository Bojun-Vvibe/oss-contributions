# google-gemini/gemini-cli PR #26358 — feat(cli): exit shell mode with backspace on empty input

- PR: https://github.com/google-gemini/gemini-cli/pull/26358
- Head SHA: `7186b7fca7c6e9a84cf5755e1de108b4809396f3`
- Author: @shkuls
- Closes: #26299
- Size: +14 / -2

## Summary

Adds backspace-to-exit symmetry for shell mode: `!` on an empty buffer enters shell mode, so backspace on an empty buffer should leave it. Also updates the `ShellModeIndicator` text from `(esc to disable)` to `(esc or backspace to disable)` and adjusts the matching test.

## Specific references from the diff

- `packages/cli/src/ui/components/InputPrompt.tsx:901-911` — new keyboard handler block. Guards on `keyMatchers[Command.DELETE_CHAR_LEFT](key)`, `buffer.text === ''`, `shellModeActive`, `!reverseSearchActive`, and `!commandSearchActive`. On match: `setShellModeActive(false)`, `resetTurnBaseline()`, return `true`. Placed *immediately above* the existing `key.sequence === '!' && buffer.text === ''` enter-shell-mode handler at line 913, which is the right neighborhood — both handlers operate on empty buffer + a single key.
- `packages/cli/src/ui/components/ShellModeIndicator.tsx:15` — indicator text update.
- `packages/cli/src/ui/components/ShellModeIndicator.test.tsx:15` — assertion bumped to `'esc or backspace to disable'`.

## Verdict: `merge-after-nits`

Symmetric, well-scoped, and the indicator text + test were updated together (the most common omission on this kind of UX tweak). The behavior table in the PR description matches the implementation.

## Nits / concerns

1. **Missing handler test.** The diff adds an `InputPrompt` behavior change but no `InputPrompt.test.tsx` case. The two interesting paths to lock down:
   - empty buffer + shell mode + backspace → `setShellModeActive(false)` is called, `resetTurnBaseline()` is called, handler returns `true`.
   - non-empty buffer + shell mode + backspace → falls through (existing delete-char behavior runs), handler does *not* call `setShellModeActive`.
   The existing test infrastructure already mocks key events; one new `it()` would do it.
2. **`reverseSearchActive` / `commandSearchActive` guards are correct but undocumented.** The PR body's behavior table doesn't mention either mode. If the user is in shell mode *and* opens reverse search, backspace there should delete characters in the search query, not exit shell mode. The guards encode this but aren't exercised by any test.
3. **`resetTurnBaseline()` call** — make sure this matches what the existing `esc` handler for shell-mode-exit does. If `esc` doesn't reset the turn baseline, this PR introduces an asymmetry where `esc` and `backspace` exit shell mode with different state side-effects. Worth a 30-second diff against the `esc`-handler block.
4. **The condition order matters for short-circuit.** `keyMatchers[Command.DELETE_CHAR_LEFT](key) && buffer.text === '' && shellModeActive && ...` — `keyMatchers` is presumably cheap (function call), but `buffer.text === ''` and `shellModeActive` are O(1) field reads. Reordering to `shellModeActive && buffer.text === '' && keyMatchers[...]` would short-circuit the keymatcher call on every non-shell-mode keystroke (which is most of them). Micro-optimization, but `InputPrompt` is on the hot path for every keystroke.
5. **Indicator text is now longer** — on narrow terminals `(esc or backspace to disable)` may wrap or get truncated where `(esc to disable)` did not. Worth a quick visual check at 60-column width.
