# BerriAI/litellm#26955 — Add Gemma 4 26B pricing for Vertex AI

- **PR**: https://github.com/BerriAI/litellm/pull/26955
- **Head SHA**: `c979d437018682bfe84811e0c7ec5e34ecbdeea2`
- **Files**: `litellm/model_prices_and_context_window_backup.json` (+13/-0), `model_prices_and_context_window.json` (+13/-0)
- **Verdict**: **merge-as-is**

## Context

Adds pricing entry for `vertex_ai/google/gemma-4-26b-a4b-it-maas` to both the in-tree pricing JSON and its backup. Per PR body: input `$0.15/M tokens` (`1.5e-07`), output `$0.60/M tokens` (`6e-07`), context 256K tokens. Source explicitly cited: `https://cloud.google.com/vertex-ai/generative-ai/pricing`. Author confirms no code changes required because `cost_router()` in `cost_calculator.py` already routes any model with `gemma` to token-based pricing, `VertexGemmaConfig` handles the `vertex_ai/gemma/*` format, and model routing in `common_utils.py` matches the `gemma/` prefix.

## What's right

- **Both files updated symmetrically.** Diff at `model_prices_and_context_window_backup.json:32735-32747` and `model_prices_and_context_window.json:32789-32801` is identical: same key, same numeric values, same field set. The two-file rule is enforced (these JSONs drift apart and break runtime behavior if only one is updated).
- **Field set matches surrounding entries.** Includes `input_cost_per_token`, `output_cost_per_token`, `litellm_provider: vertex_ai`, `max_input_tokens: 256000`, `max_output_tokens: 8192`, `max_tokens: 8192`, `mode: chat`, `source: <URL>`, `supports_function_calling: true`, `supports_system_messages: true`, `supports_vision: true`. Same shape as the adjacent `vertex_ai/deep-research-pro-preview-12-2025` block at `:32750+`.
- **Source URL embedded in the entry itself**, not just the PR body. This is the right discipline — the next person to update this entry can immediately go check the source for changes without spelunking through PR history.
- **No code changes claimed in PR body and verified by the diff** — the diff is purely the two JSON insertions. The claim that existing routing handles this model is consistent with the surrounding `vertex_ai/google/gemma-*` keys.
- **Numeric format is consistent.** `1.5e-07` and `6e-07` use the same scientific-notation shape as adjacent entries. No floating-point divergence.

## Risks / nits

- **`max_input_tokens: 256000` should be confirmed against the Vertex page** — many Gemma variants ship with 8K context; 256K would be a significant capability shift. PR body cites the pricing page, but worth a sanity check at merge time.
- **Optional: a quick assertion that the model is also queryable via the existing `vertex_ai/gemma/*` prefix routing** — i.e., that `cost_router("vertex_ai/google/gemma-4-26b-a4b-it-maas")` returns the new entry rather than an older `vertex_ai/gemma-26b` fallback. Could be a one-line CI test but not a merge blocker.
- **No `supports_vision` source citation** — the PR body says "Supports: chat, function calling, vision, system messages" but the linked Vertex pricing page doesn't typically enumerate capability matrices. If `supports_vision: true` is wrong, callers will pass image content and get a runtime error. Worth a quick verification.

## Verdict

**merge-as-is.** Pure pricing-table addition, both files updated symmetrically, source cited inline, no code changes needed because the existing routing infrastructure already covers the `gemma` prefix. The "two pricing files updated together" rule is the load-bearing discipline this PR follows correctly.
