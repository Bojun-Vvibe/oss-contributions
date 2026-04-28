# google-gemini/gemini-cli PR #26092 — fix(cli): handle DECKPAM keypad Enter sequences in terminal

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26092
- **Author**: @Gitanaskhan26
- **Head SHA**: `f05946cb815d7168cbc5a5a50ceef23f29e7e929`
- **Size**: +1 / −0 in `packages/cli/src/ui/contexts/KeypressContext.tsx`

## Summary

A one-line addition to the `KEY_INFO_MAP` in `KeypressContext.tsx` mapping the escape sequence `OM` to the canonical `enter` key. This sequence is what some terminals (notably xterm-class emulators in DECKPAM "application keypad" mode) emit when the user presses the *numeric-keypad* Enter key instead of the main Return key — without this mapping, that keystroke either does nothing or is rendered as the literal characters `OM` in the input box, depending on what downstream code does with unmatched escape sequences.

## Verdict: `merge-as-is`

The fix is exactly the right shape: a single, surgical entry in the same `KEY_INFO_MAP` table where the other keypad/cursor escape sequences live (`[A`, `[B`, `[200~`, `[[A`, etc.). It costs one line, has zero blast radius beyond "users with numeric keypads can now press Enter on them," and matches the established pattern. The `OM` sequence is well-documented in the xterm DECKPAM/DECKPNM control documentation as the application-mode keypad-Enter response, so this is not a guess.

## Specific references

- `packages/cli/src/ui/contexts/KeypressContext.tsx:44` — `OM: { name: 'enter' }` is added at the top of `KEY_INFO_MAP`. The placement above `[200~` (paste-start) is alphabetically reasonable since the leading character is `O` (uppercase) which sorts before `[` (ASCII 91 vs. 79). Trivial style choice — could just as well live next to the other Enter-shaped entries if one exists, but the file's existing organization isn't strictly grouped, so this placement is fine.
- The mapping returns only `{ name: 'enter' }` with no `shift`/`ctrl` flags. Correct: numeric-keypad Enter without modifiers should be indistinguishable from main-keyboard Return for downstream consumers, which is the user's mental model.
- The `KEY_INFO_MAP` is keyed by the *post-`ESC`* tail of the sequence (e.g. `'[200~'` covers `ESC [200~`), so `'OM'` here covers the actual wire sequence `ESC O M` (0x1b 0x4f 0x4d). This matches how DECKPAM-mode keypad keys are documented (`ESC O M` for keypad Enter, vs. `ESC [ M` which is mouse-tracking — the difference is the `O` vs `[` byte after `ESC`, so the discriminator is correct and there is no collision risk).

## Nits / follow-ups

1. While you're in DECKPAM territory, the same xterm spec defines `ESC O P` through `ESC O S` for keypad-mode F1-F4 (vs. the `ESC [[A` through `ESC [[E` linux-console mappings already present in the table). If users are reporting keypad-Enter doesn't work, they may also be hitting keypad-F1/F2/etc. silently swallowing. Worth a follow-up adding `OP`, `OQ`, `OR`, `OS` (mapped to `f1`, `f2`, `f3`, `f4`) for consistency.
2. The same spec also defines the keypad-mode arrow keys as `ESC O A/B/C/D` (vs. the cursor-mode `ESC [ A/B/C/D` already in the table at the top). On terminals that boot into application-mode by default, arrow keys may also be mis-rendered as `OA`/`OB`/etc. literals. Worth verifying the gemini-cli main-loop sends a normal-mode `ESC[?1l` reset on startup, or else add the four `OA`-`OD` entries here too.
3. No test added for this entry. The existing test file (if any) for `KeypressContext` would benefit from a parametric "every entry in `KEY_INFO_MAP` produces a key event with the declared `name`" sweep — that would lock in this entry plus catch typos in any future addition.
