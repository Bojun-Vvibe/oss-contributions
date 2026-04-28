# BerriAI/litellm #26705 — [Feat] Add gpt-image-2 support

- PR: https://github.com/BerriAI/litellm/pull/26705
- Head SHA: `958c35a8016c38f87a0057ba8f68068b667766c0`
- Author: ishaan-berri
- Files touched: `litellm/litellm_core_utils/get_llm_provider_logic.py`, `litellm/litellm_core_utils/llm_cost_calc/utils.py`, `litellm/llms/azure/image_generation/__init__.py`, `litellm/llms/azure/image_generation/gpt_transformation.py`, `litellm/llms/openai/image_generation/cost_calculator.py`, `litellm/llms/openai/image_generation/gpt_transformation.py`, `litellm/model_prices_and_context_window_backup.json`, `litellm/utils.py`, `model_prices_and_context_window.json`, `tests/test_litellm/test_gpt_image_cost_calculator.py`, `tests/test_litellm/test_utils.py`

## Observations

- `litellm/litellm_core_utils/get_llm_provider_logic.py:351` — adds `or model.startswith("gpt-image")` to the OpenAI auto-detection chain. This is the right widening: it future-proofs detection beyond `gpt-image-1` so any `gpt-image-*` family routes to the OpenAI provider without an explicit registry hit.
- `litellm/litellm_core_utils/llm_cost_calc/utils.py:986-988` and `:1008-1010` — both the OpenAI and Azure branches change `if "gpt-image-1" in model_lower:` to `if "gpt-image" in model_lower:`. Functionally equivalent for `gpt-image-1` and forward-compatible with `gpt-image-2`. Also matches the comment cleanup ("gpt-image models use token-based pricing").
- `litellm/llms/azure/image_generation/__init__.py:27` — log line updated from "follows the gpt-image-1 model format" to "follows the gpt-image model format". Cosmetic but honest about the broadened match.
- Tests are added in `tests/test_litellm/test_gpt_image_cost_calculator.py` and `tests/test_litellm/test_utils.py` — good coverage discipline.
- Slight concern: `model.startswith("gpt-image")` is broad. If a future provider ships `gpt-image-classifier` or similar non-generation model, it would be misrouted. A regex like `^gpt-image-\d` would be tighter but is bikeshed-tier.

## Verdict: `merge-after-nits`

**Rationale:** Clean broadening of pattern matching for the next gpt-image generation. Pricing tables and provider routing all updated together. Only nit: consider tightening `startswith("gpt-image")` to a numbered-suffix pattern, otherwise this lands cleanly.
