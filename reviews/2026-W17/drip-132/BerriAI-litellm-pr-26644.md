# BerriAI/litellm PR #26644 — Add gpt-image-2 support

- **PR**: https://github.com/BerriAI/litellm/pull/26644
- **Author**: @emerzon
- **Head SHA**: `c7d46971c8b1c36753041feff79a6c227a1199d6`
- **Size**: +298 / −11 across 11 files

## Summary

Adds support for the `gpt-image-2` family in two surfaces: (1) provider detection — both `get_llm_provider` and `validate_environment` now recognize any `model.startswith("gpt-image")` string as the OpenAI provider; (2) cost routing — the `gpt-image-1`-specific substring guards in `route_image_generation_cost_calculator` are widened to `"gpt-image" in model_lower` so future revisions (`-2`, `-3`, ...) automatically route to the token-based calculator instead of falling through to the pixel-based DALL-E path. Adds four registry entries (`gpt-image-2`, `gpt-image-2-2026-04-21`, and the two `azure/`-prefixed siblings) with token-based pricing in both `model_prices_and_context_window.json` and the `_backup.json` copy. Adds three end-to-end tests pinning the routing and cost math for the new entries.

## Verdict: `merge-after-nits`

The shape is right — the change from a hard-coded version-suffix check (`gpt-image-1`) to a family-prefix check (`gpt-image`) is the correct generalization for what is clearly going to be a multi-version family. The pricing-registry entries are symmetrically applied across both files (no risk of the load-bearing primary registry and the bundled backup falling out of sync), and the test matrix at `tests/test_litellm/test_utils.py:213-279` covers the new family-prefix entry, the dated snapshot version, and the Azure-prefixed entry as three independent cells. But three concerns deserve attention.

## Specific references

- `litellm/litellm_core_utils/get_llm_provider_logic.py:351` — `model.startswith("gpt-image")` is added to the OpenAI provider-detection chain. This is the load-bearing change; without it, any non-registered `gpt-image-*` string would fall through to the unknown-provider error path. Correct shape and matches the symmetric addition in `litellm/utils.py:6529`. **But** there is no symmetric prefix for the Azure case — `azure/gpt-image-2` is detected by registry-key match in the same file via the existing Azure-prefix logic, not by `startswith` here. Worth confirming that the Azure detection path falls back gracefully when an unregistered `azure/gpt-image-3-preview` string appears in the future; if not, the symmetric Azure prefix also needs adding.

- `litellm/litellm_core_utils/llm_cost_calc/utils.py:985` and `:1007` — both `OPENAI` and `AZURE` arms widen `"gpt-image-1" in model_lower` to `"gpt-image" in model_lower`. Symmetric and correct. The substring check is technically also true for a hypothetical `"my-gpt-image-classifier"` model name, but registry-resident model strings are controlled and the false-positive risk is low. Acceptable.

- `litellm/model_prices_and_context_window_backup.json:5104-5145` and `:19115-19146` — four new entries with identical `cache_read_input_image_token_cost: 2e-06`, `input_cost_per_image_token: 8e-06`, `output_cost_per_image_token: 3e-05`. The pricing values themselves can't be sanity-checked from inside the diff; they should be cross-referenced against the OpenAI/Azure pricing pages before merge. The `gpt-image-2-2026-04-21` snapshot entry duplicates the bare-`gpt-image-2` entry exactly, which is the right shape for a "current alias" + "pinned snapshot" pair.

- `tests/test_litellm/test_utils.py:215-220` — `test_gpt_image_provider_detection_covers_existing_family` is the right kind of regression anchor — it asserts the old `gpt-image-1`, `gpt-image-1-mini`, `gpt-image-1.5` are still detected, so a future "let's narrow the prefix" change fails this test loudly. **Strong**: this is the kind of test that locks in the contract change rather than just exercising the new path.

- `tests/test_litellm/test_gpt_image_cost_calculator.py:32-45` — new `_use_local_model_cost_map` autouse fixture ensures tests use the in-repo registry rather than fetching from the live LiteLLM cost map URL. Correct hermeticity choice.

- `tests/test_litellm/test_gpt_image_cost_calculator.py:165-194` — `test_gpt_image_2_cost_with_text_and_image_tokens` pins the four-component cost math (text-input + image-input + text-output + image-output) with literal arithmetic in the comment block, which is exactly the right shape for catching a "we accidentally dropped the image-output multiplier" regression. **Strong**.

## Nits

1. The tests don't cover the cache-read paths (`cache_read_input_image_token_cost`, `cache_read_input_token_cost`). The pricing-registry entries declare these fields, but if the cost calculator doesn't actually consult them, the registry data is dead. Add a `prompt_tokens_details` with `cached_tokens > 0` and assert the cache-discounted cost.
2. The `model_lower = model.lower()` at `:984` and `:1006` then `if "gpt-image" in model_lower` accepts any case. The OpenAI canonical naming is lowercase, but this means `"GPT-IMAGE-2"` from a misconfigured upstream call would also route. Probably fine but worth a one-line comment naming the case-insensitivity as deliberate.
3. `litellm/llms/openai/image_generation/cost_calculator.py:18` docstring updated to "gpt-image family" but the example in the `Args:` block at `:24` shows `"gpt-image-1", "gpt-image-2"`. Once you add `gpt-image-3` next year, this docstring will go stale; consider just "gpt-image-N" instead.
4. The pricing-registry entries declare `"supports_pdf_input": true` for an image-generation model. If `gpt-image-2` truly accepts PDF input as a *prompt* (not just as a multi-page image source), worth a one-line comment in the JSON noting why an image-generation model has a PDF-input flag. If not, this is a copy-paste artifact from `gpt-image-1.5` and should be removed.
