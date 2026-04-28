---
pr_number: 26092
repo: google-gemini/gemini-cli
head_sha: 4a2ae68dd70fcde22f36efe1659935aab8a0ca4e
verdict: merge-after-nits
date: 2026-04-28
---

# google-gemini/gemini-cli#26092 — fix(cli): handle DECKPAM keypad Enter sequences in terminal

**What changed.** Single-line +1/−0 in `packages/cli/src/ui/contexts/KeypressContext.tsx:44`: adds `'OM': { name: 'enter' }` to `KEY_INFO_MAP`. Closes #22671.

**Why it matters.** When a terminal is in DECKPAM (Application Keypad Mode) — which VS Code's Linux integrated terminal enables by default — the numeric keypad Enter key emits `\eOM` instead of `\r` or `\n`. The existing `NUMPAD_MAP` covers numeric keypad digits (which mostly map to text characters), but `\eOM` for Enter has no text-character fallback so it dropped entirely. Users hit Numpad-Enter in VS Code's Linux terminal and nothing happened.

**Concerns.**
1. **Stripped `\eO` prefix** — the existing key-decode pipeline must already be peeling off the leading `\e` (ESC) and `O` (SS3 single-shift) sequence indicator and dispatching the remaining `M` against `KEY_INFO_MAP`. Adding `'OM'` (not `'M'`) suggests the dispatcher key is the post-ESC, pre-final-byte slice. Verify the existing `OM` lookup actually matches what the decoder produces — if the decoder strips only `\e` and dispatches `'OM'` against the map, this is correct; if it dispatches `'M'`, this entry never fires.
2. **`KEY_INFO_MAP` already contains `'[200~'` / `'[201~'` / `'[[A'` / etc.** — those follow the CSI (`\e[`) encoding, not SS3 (`\eO`). The `OM` entry sits next to them as the first SS3-prefixed entry. Worth a one-line comment block above the `OM` entry naming "SS3 (`\eO`)-prefixed keypad sequences" so a future reader doesn't try to extend with bare `'M'` thinking it's the same dispatch class.
3. **Other DECKPAM SS3 sequences exist** that this PR doesn't address: `\eOP/Q/R/S` (F1-F4 in some terminals — separately from CSI-encoded `[[A-D` already in the map), `\eOj/k/l/m/n/o/p/q/r/s/t/u/v/w/x/y/M` (full numpad in DECKPAM mode: `*`/`+`/`,`/`-`/`.`/`/`/`0`-`9`/Enter). Right now the PR plugs Enter only. Any user who reported #22671 will likely also report numeric digit drops on the same VS Code Linux terminal once the workaround for Enter ships and they keep using the keypad. Recommend either (a) widening to the full DECKPAM map in this PR, or (b) explicitly noting in the PR body that other DECKPAM digit sequences may need follow-up — current PR body claims "Other numeric keypad characters were correctly mapped via `NUMPAD_MAP`" which is true *only when DECKPAM is off* (digits emit text characters then), false *when DECKPAM is on* (digits emit `\eOq` etc., not text).
4. **No test added.** Even a minimal cell in `KeypressContext.test.tsx` (or wherever the dispatcher is tested) feeding `\x1bOM` and asserting `name: 'enter'` would prevent regression on the next refactor of `KEY_INFO_MAP` ordering or the decode pipeline. PR checklist "Added/updated tests" is checked but the diff shows zero test additions.
5. **Cross-platform validation matrix in PR body** has only Linux checked; macOS Terminal.app and Windows Terminal both can request DECKPAM via app-side CSI sequences too, so the fix should benefit them equally — but verify the underlying decoder dispatches the same `'OM'` string on those platforms (Windows ConPTY in particular has had historical quirks with SS3 sequences).
6. **`{ name: 'enter' }` without `shift`/`ctrl`** is correct — DECKPAM Enter has no modifier-bit encoding in the SS3 form (modified Enter goes through CSI `\e[27;...M` or similar), so the bare `enter` mapping matches what's emitted.

Surgical one-line fix to a real bug. Ship after either widening to full DECKPAM keypad coverage or adding a follow-up issue (concern 3) and a one-cell decode test (concern 4).
