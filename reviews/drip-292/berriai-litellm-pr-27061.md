# BerriAI/litellm PR #27061 — Add Azure AI MAI image generation support

- **Repo:** BerriAI/litellm
- **PR:** #27061
- **Head SHA:** `e4172c2e93e02d221bf788cf7e0f27301d741209`
- **Author:** emerzon
- **Title:** Add Azure AI MAI image generation support
- **Diff size:** +596 / -17 across 8 files
- **Drip:** drip-292

## Files changed

- `litellm/llms/azure_ai/image_generation/mai_transformation.py` (+198/-0) — new `AzureFoundryMAIImageGenerationConfig` config + transport.
- `litellm/llms/azure_ai/image_generation/__init__.py` (+33/-7) — adds `_supports_mai_endpoint(model)` lookup and selects the MAI config when the model entry has `supports_mai_endpoint: true`.
- `litellm/llms/azure_ai/image_generation/cost_calculator.py` (+50/-10) — splits cost calc into usage-based (token-priced) vs. per-image fallback via `_calculate_cost_from_usage`.
- `litellm/images/main.py` (+23/-0) — adds an MAI-specific `image_generation_handler` short-circuit branch when the resolved config is `AzureFoundryMAIImageGenerationConfig`.
- `model_prices_and_context_window.json` (+38/-0) and `litellm/model_prices_and_context_window_backup.json` (+38/-0) — catalog entries for `MAI-Image-2` and `MAI-Image-2e`.
- `litellm/types/llms/openai.py` (+2/-0) — extends `OpenAIImageGenerationOptionalParams`.
- `tests/test_litellm/llms/azure_ai/image_generation/test_azure_ai_mai_image_generation_metadata.py` (+214/-0) — new metadata + transport tests.

## Specific observations

- `image_generation/__init__.py:18-30` (`_supports_mai_endpoint`) — wraps `litellm.get_model_info` in a bare `except Exception` and falls through to the old normalize-and-substring-match path. Reasonable defensive default, but the `verbose_logger.debug` swallows the underlying error type; for an integrators debugging "why isn't my MAI model routing", a `verbose_logger.debug("...: %s", exc)` would save real time.
- `image_generation/__init__.py:34-58` — the legacy normalize/substring branch (`"dalle2" in normalized_model`, `"flux" in normalized_model`) is now subordinate to the MAI lookup. That's the right precedence, but make sure the MAI catalog entries set `supports_mai_endpoint: true` for *every* MAI variant; if a future MAI-Image-3 ships and the JSON forgets the flag, it will silently fall through to `AzureFoundryGPTImageGenerationConfig` rather than failing loudly.
- `cost_calculator.py:31-39` (`cost_calculator`) — moved the `isinstance` guard to the top before any model lookups. Good — removes a redundant `get_model_info` call on the failure path. The new docstring is accurate.
- `cost_calculator.py:46-72` (`_calculate_cost_from_usage`) — the early return when `prompt_tokens == 0 and completion_tokens == 0 and total_tokens == 0` is correct for "model returned a usage object but didn't bill anything", but treating `None` and "all zeros" the same way will hide regressions where a provider starts emitting zero tokens for a billable image. Worth a `verbose_logger.debug` when zeros are seen.
- `cost_calculator.py:66-71` — `output_cost_per_image_token or output_cost_per_token or 0.0` chain: the fallback to `output_cost_per_token` is generous and could double-bill if a model has both keys for different reasons. Confirm intentional.
- `images/main.py:454-481` — the new MAI branch runs `litellm_params_dict["api_base"] = api_base` *before* the isinstance check, then dispatches to `llm_http_handler.image_generation_handler` and `return`s. The non-MAI path below still uses the old `default_headers` flow. The early-return cleanly avoids cross-contamination — good shape.
- `mai_transformation.py:1-12` — module-level imports include `urllib.parse.parse_qsl, urlencode, urlsplit, urlunsplit`; reviewer should confirm these are all used (the diff truncates before the body). If only `urlsplit/urlunsplit` are needed, trim the import.
- Pricing entries in `model_prices_and_context_window.json` are duplicated in `model_prices_and_context_window_backup.json` — this is the existing repo convention for that file but worth flagging that both must stay in sync in any follow-up.

## Verdict: `merge-after-nits`

The architecture is right: model-flag-driven dispatch beats string sniffing, the cost path is properly split, and the early-return in `images/main.py` keeps the MAI flow isolated. Address the silent-fallback debug logging in `_supports_mai_endpoint`, add a debug log for the all-zero usage case, and confirm the `output_cost_per_image_token` / `output_cost_per_token` fallback is intentional.
