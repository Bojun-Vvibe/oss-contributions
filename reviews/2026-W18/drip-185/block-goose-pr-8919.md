---
pr: block/goose#8919
sha: 6e1f39031572ab95b1e09ff2a3ec728d96f552df
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# feat(ui): add codex intelligence selector

URL: https://github.com/block/goose/pull/8919
Files: `ui/desktop/src/components/settings/models/subcomponents/SwitchModelModal.tsx`
Diff: 130+/13-

## Context

The switch-model modal already exposes "Claude thinking effort"
(`disabled/low/medium/high/max`) for Claude-family models. Equivalent
selector for the `chatgpt_codex` provider's reasoning-effort dial
(`low/medium/high/xhigh`) was missing — users had to hand-edit
`CHATGPT_CODEX_REASONING_EFFORT` config or live with the default. PR
adds a parallel UI gate keyed off the provider name plus a
model-specific carve-out for the `xhigh` tier.

## What's good

- `isChatGptCodexProvider(name)` at `:224-226` and the modal-level
  `showChatGptCodexIntelligence = isChatGptCodexProvider(selectedProvider)`
  at `:355` is exactly the right shape: the gate keys off
  `selectedProvider` (which respects the `usePredefinedModels` ternary
  at `:351`), not the model name, since reasoning-effort is a
  provider-level concept here.
- `supportsExtraHighChatGptCodexIntelligence` at `:228-230` allowlists
  `gpt-5.4` and `gpt-5.3-codex` — the only models that accept the
  `xhigh` tier today. The downstream filter at `:357-362` falls back
  to `['medium', 'high']` for any other model, defensively avoiding a
  "user picks xhigh on a model that 400s the request" path.
- The `selectedChatGptCodexReasoningEffort` clamp at `:363-368` is
  load-bearing: if the user previously selected `xhigh` then switched
  to a model that doesn't support it, the persisted state would
  otherwise survive into a request that fails — the clamp coerces
  back to `medium` whenever the persisted value isn't in the
  visible-options set, mirroring the same pattern used for Claude
  thinking effort.
- i18n done correctly at `:48-65,177-187` — every new label string is
  routed through `defineMessages` with stable IDs (`switchModelModal.codexIntelligence*`)
  rather than inlined, so the four tier labels and the section header
  can be translated without a follow-up sweep.
- Default tier `medium` at `:354` matches both the OpenAI Codex CLI
  and Claude-thinking defaults for parity — users moving between
  providers see consistent baseline behaviour.

## Nits

- `isChatGptCodexProvider` uses a strict equality check `name ===
  'chatgpt_codex'` while `isClaudeModel` uses a `startsWith` prefix.
  Consistent pattern would be `name?.toLowerCase() === 'chatgpt_codex'`
  — guards against a future config where the provider key is
  `ChatGPT_Codex` or `chatgpt-codex`. Same consideration for
  `supportsExtraHighChatGptCodexIntelligence` which `===`-matches two
  exact model strings.
- `supportsExtraHighChatGptCodexIntelligence` hardcodes
  `gpt-5.4`/`gpt-5.3-codex`. The next reasoning-capable Codex model
  will silently lose `xhigh` until someone updates this allowlist.
  Lift to a module-level `XHIGH_SUPPORTED_CODEX_MODELS = new Set([...])`
  and add a comment naming the decision criterion ("models with
  documented reasoning-effort: xhigh tier in OpenAI API").
- The visible-options filter at `:359-362` mutates the displayed list
  but the underlying `chatGptCodexReasoningEffort` state can still be
  set to `xhigh` for an unsupported model (the clamp only affects
  what's *passed downstream*, not what's stored). Net: re-selecting
  the original model would re-show `xhigh` as selected. Probably
  intentional UX (preserve user's preferred level when they go back
  to a supporting model) but worth a comment so the next reader
  doesn't "fix" it.
- No test coverage for the new options array or the visibility/clamp
  predicates. A unit test on `supportsExtraHighChatGptCodexIntelligence`
  + the filter logic at `:357-362` would lock the model-allowlist
  behaviour. Currently the entire feature relies on manual visual
  verification.
- The persistence path (where this gets written back to config) isn't
  visible in this diff range. Verify there's a corresponding
  `upsert('CHATGPT_CODEX_REASONING_EFFORT', selectedChatGptCodexReasoningEffort)`
  call elsewhere in the modal — without it, the selector is
  display-only and resets on every modal open.
- The boundary `'gpt-5.4'` may collide with `'gpt-5.4-mini'` /
  `'gpt-5.4-turbo'` etc. in the future. `===` is safe today but if
  the allowlist moves to `startsWith`/`includes` for
  forward-compatibility, the substring collision risk emerges.

## Verdict reasoning

Correctly-shaped UI feature mirroring an existing in-codebase pattern
(`isClaudeModel`/`supportsAdaptiveThinking` → `isChatGptCodexProvider`/
`supportsExtraHighChatGptCodexIntelligence`). i18n done right, default
tier matches sibling providers, the `selectedChatGptCodexReasoningEffort`
clamp prevents the "persisted xhigh on a non-supporting model" 400
class. Nits are: provider-name case sensitivity, model allowlist
hardcoded inline, missing unit test on the visibility predicates,
and need to verify the config-persistence callsite. None blocking;
merge after either lifting the allowlist to a constant or adding a
predicate test.
