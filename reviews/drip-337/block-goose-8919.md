# block/goose #8919 — feat(ui): add codex intelligence selector

- **Head SHA reviewed:** `aa4bcda070a9786ae493fa90663556f2e4ac7ba3`
- **Size:** +142 / -13 across 2 files
- **Verdict:** merge-after-nits

## Summary

Adds an "Intelligence" dropdown to the desktop SwitchModelModal for
the `chatgpt_codex` provider. Persists the selection to the
`CHATGPT_CODEX_REASONING_EFFORT` global config key. Levels:
`low`, `medium`, `high`, plus `xhigh` for known top-tier model
names (`gpt-5.4`, `gpt-5.3-codex`).

## What I checked

- `ui/desktop/src/components/settings/models/subcomponents/SwitchModelModal.tsx:225-229`
  — `supportsExtraHighChatGptCodexIntelligence` hardcodes two model
  name strings (`'gpt-5.4'`, `'gpt-5.3-codex'`). This is brittle:
  the moment a new GPT-5.x or successor lands, users will silently
  miss the `xhigh` option. Move the allowlist to a typed constant
  at module top, or better, derive it from a server-side capability
  flag the way `supportsAdaptiveThinking` could.
- `SwitchModelModal.tsx:359-362` — reads `CHATGPT_CODEX_REASONING_EFFORT`
  and gates assignment on
  `!userSelectedChatGptCodexReasoningEffort.current`. The ref guard
  is the correct pattern to avoid clobbering an in-flight user
  selection during async config load. Good.
- `SwitchModelModal.tsx:367-377` — `visibleChatGptCodexIntelligenceOptions`
  filters to `['medium', 'high']` for unknown models. But the
  initial state is `'medium'`, which means a user who *had*
  previously persisted `'low'` against, say, `gpt-5.2` will see the
  selector silently snap to `'medium'`. The
  `selectedChatGptCodexReasoningEffort` derived value handles this
  ("if not in visible options, fall back to medium"), but it doesn't
  *write* the corrected value back, so the persisted config will
  drift from the displayed UI. Either:
  - write the corrected value to config on detection, or
  - add a one-time toast warning the user.
- `SwitchModelModal.tsx:463-474` — the `upsert` happens unguarded
  on every modal save when `showChatGptCodexIntelligence` is true,
  even if the user didn't touch the selector. That's harmless if
  the value matches what's already stored, but it does generate
  unnecessary config writes. Wrap in
  `if (userSelectedChatGptCodexReasoningEffort.current)` for
  symmetry with how Claude effort is handled (or check current vs.
  pending).
- `ui/desktop/src/i18n/messages/en.json` (claimed in files list) —
  ensure the four new `i18n.codexIntelligence*` IDs landed in the
  i18n source file, otherwise extraction CI will flag them.
- The function name `isChatGptCodexProvider` checks
  `name === 'chatgpt_codex'` — fine for now, but if Goose ever adds
  a second variant of the same provider (e.g., `chatgpt_codex_eu`),
  this string compare breaks. Switch to `startsWith`.

## Risk

Low. UI-only, additive, gated to a single provider, default value
matches prior behavior (medium).

## Recommendation

Merge after:

1. Lift the `gpt-5.4` / `gpt-5.3-codex` allowlist into a named
   constant at the top of the file (or pull from server capability
   metadata).
2. Decide what to do when persisted effort is invalid for the
   current model (write-correct vs. warn vs. silently snap).
3. Skip the `upsert` when the user didn't actually change the
   selector, mirroring the Claude path.
4. Verify the i18n extraction picked up all four new message IDs.
