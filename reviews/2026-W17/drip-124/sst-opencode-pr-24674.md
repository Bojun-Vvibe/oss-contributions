# sst/opencode #24674 — fix(tui): preserve terminal selection when copy-on-select is disabled

- **PR**: [sst/opencode#24674](https://github.com/sst/opencode/pull/24674)
- **Head SHA**: `ba5f4c70`
- **Closes**: #5046

## Summary

One-line guard added to the `renderer.console.onCopySelection` callback at
`packages/opencode/src/cli/cmd/tui/app.tsx:296`: when
`Flag.OPENCODE_EXPERIMENTAL_DISABLE_COPY_ON_SELECT` is set, return early so the
opentui callback doesn't auto-copy and clear the selection. This restores
terminal-native selection for users who rely on tmux/Windows Terminal copy
gestures (e.g. mouse drag → Ctrl+Shift+C in Windows Terminal, or
`copy-pipe-and-cancel` in tmux), where opencode's auto-copy was racing the
terminal's own selection-buffer commit and visually clearing the highlight
mid-drag.

## Specific findings

- `app.tsx:296` — guard placement is correct: it sits *after* the empty-text
  early-return at `:295` (so unrelated callback noise from spurious empty
  selection events is still filtered out at the front) and *before* the
  `Clipboard.copy(text)` call. Net effect: when the flag is set, the callback
  becomes a no-op for non-empty selections, which is the actual desired behavior
  ("opentui hands us the selection text, we throw it away and let the terminal
  handle the actual buffer copy").
- The flag name is consistent with the existing `Flag.*` namespace and carries
  the `OPENCODE_EXPERIMENTAL_` prefix that signals "behavior may change without
  a deprecation cycle" — appropriate for a workaround that depends on opentui's
  internal selection-event semantics.
- The check reads the flag at every callback invocation rather than caching it
  at `renderer.console.onCopySelection` assignment time. This is the right
  choice for an env-var-backed flag (it picks up live config reload paths and
  doesn't bake in startup state), and the cost is one boolean read per copy
  event — well under the threshold where it would matter.

## Nits / what's missing

- No regression test. The opentui `console.onCopySelection` callback isn't
  trivially mockable without refactoring, but a minimal unit test that
  toggles the flag and asserts `Clipboard.copy` isn't called when set would
  pin the contract — currently a future refactor that swaps the
  early-return order or accidentally moves the flag check below the
  `Clipboard.copy` call would land silently.
- PR body cites Windows Terminal and tmux as the motivating cases; the flag
  itself doesn't document either in its inline help / `Flag` declaration.
  Worth a one-line comment naming the load-bearing terminal-emulator
  behavior the flag exists to accommodate.

## Verdict

**merge-as-is** — surgical 1-line guard at the exact callback site, correct
ordering relative to the existing empty-text early-return, opt-in via an
explicitly experimental flag so non-flag users see no change. Nits are real
but don't block the fix.
