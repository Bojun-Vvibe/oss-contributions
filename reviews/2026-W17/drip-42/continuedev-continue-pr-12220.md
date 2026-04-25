# continuedev/continue PR #12220 — feat: add FuturMix as a model provider

- **URL:** https://github.com/continuedev/continue/pull/12220
- **Head SHA:** `a9b7db2a8487d415f065a441cd21afcd14135b69`
- **Files touched:** 6 (provider class, provider registry, docs, logo asset, model configs, provider configs)
- **Verdict:** `request-changes`

## Summary

Adds FuturMix (an OpenAI-compatible aggregator gateway) as a first-
class provider. Subclasses the existing `OpenAI` LLM with a fixed
`apiBase`, registers it in the LLM index, ships an MDX docs page, and
wires GUI add-model entries.

## Specific references

- `core/llm/llms/FuturMix.ts` (new, +13) — minimal subclass that sets
  `static providerName = "futurmix"` and `apiBase =
  "https://futurmix.ai/v1/"`. Mirrors how `Tensorix`, `SiliconFlow`,
  etc. are wired.
- `core/llm/llms/index.ts:127-130` — adds the import and pushes
  `FuturMix` into `LLMClasses`. Correct slot.
- `gui/src/pages/AddNewModel/configs/models.ts` (+94) and
  `providers.ts` (+38) — declares 22 models and the provider card.
- `docs/customize/model-providers/more/futurmix.mdx` (+97) — docs page
  recommending `claude-sonnet-4-6` as the default chat model.

## Reasoning

The mechanical wiring is fine and follows the existing aggregator
pattern. The reason for `request-changes` is upstream policy and
verifiability:

1. Continue's contribution guide for new providers asks for either an
   official endorsement from the provider or evidence of meaningful
   user demand. There's no issue link, no community thread cited.
2. The docs page hard-codes specific upstream model names
   (`claude-sonnet-4-6`, etc.) that belong to OpenAI / Anthropic /
   Google. If FuturMix's catalog drifts, this docs page silently
   rots — at minimum link to FuturMix's live model list rather than
   pinning 22 model IDs in `gui/.../models.ts` for a third-party
   gateway whose lineup the upstream cannot guarantee.
3. The new logo `gui/public/logos/futurmix.png` (binary) lacks any
   licensing/attribution note in the PR description.

Ask the author to (a) link a tracking issue or community ask, (b)
trim `models.ts` to a minimal set or generate it dynamically, and (c)
confirm logo licensing. The 13-line provider class itself is fine.
