# openai/codex #20282 — tui: return from side chat on Ctrl-D

- **URL:** https://github.com/openai/codex/pull/20282
- **Head SHA:** `dc285ff3f5c32ffacfa2e641d8cfe51d858514e5`
- **Files:** `codex-rs/tui/src/app/platform_actions.rs` (+11/-3)
- **Verdict:** `merge-as-is`

## What changed

`side_return_shortcut_matches` at `platform_actions.rs:65-75` now matches both Ctrl-C *and* Ctrl-D as the side-chat return shortcut. The match guard becomes `modifiers.contains(KeyModifiers::CONTROL) && (c.eq_ignore_ascii_case(&'c') || c.eq_ignore_ascii_case(&'d'))`. The corresponding inline test at `:78-110` is renamed `side_return_shortcuts_match_esc_ctrl_c_and_ctrl_d` and the previous *negative* assertion `assert!(!side_return_shortcut_matches(... Char('d'), CONTROL ...))` is flipped to a *positive* assertion, plus an explicit uppercase `Char('D')` case is added to lock the case-insensitive behavior.

## Why it's right

- One-line behavioral expansion, contract change is clearly intentional and correct: Ctrl-D is the conventional "leave / EOF" key in TUIs (mirrors readline, vim, less). Treating it identically to Ctrl-C for *side-chat return* is the right shape — it doesn't conflict with Ctrl-D's meaning anywhere else in the side-chat surface (no input buffer to "EOF" out of).
- The test flip is the audit-trail this needs. The previous version explicitly *asserted* Ctrl-D should NOT match — the inverted assertion at `:100-103` is the test saying "the contract changed and we mean it", not a stealth behavior shift.
- `eq_ignore_ascii_case` covers both `'d'` and `'D'`, which matters when the terminal emits an uppercase keysym depending on Shift state. The new `Char('D')` case at `:106-109` locks that.
- Surface area: one match arm, one test rename, one assertion flip, one assertion add. Clean.

## Nits

- None worth blocking on. A drive-by could add `Char('q')` with `KeyModifiers::NONE` if the broader UX wants a non-modifier exit, but that's an unrelated UX call.

## Risk

Negligible. The only collision risk would be if some other side-chat handler swallows Ctrl-D before this matcher runs — verify the dispatcher order, but the test is the canary if anyone re-routes.
