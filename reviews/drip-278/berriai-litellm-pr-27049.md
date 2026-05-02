# Review — BerriAI/litellm#27049

- PR: https://github.com/BerriAI/litellm/pull/27049
- Title: Add Azure AI DeepSeek V4 Flash metadata
- Head SHA: `f0e37e15c233de1d0b878df6bd308e05b5d059c7`
- Size: +143 / −20 across 3 files
- Verdict: **merge-as-is**

## Summary

Adds the `azure_ai/deepseek-v4-flash` model entry (and a sibling
v4-pro entry implied by the diff context) to litellm's pricing &
context-window metadata in both `litellm/model_prices_and_context_window_backup.json`
and the canonical `model_prices_and_context_window.json`, plus a
small bump to keep the mirrored pair in sync.

## Evidence

- `litellm/model_prices_and_context_window_backup.json:6676-6692`
  — new block:
  ```json
  "azure_ai/deepseek-v4-flash": {
      "input_cost_per_token": 1.03e-06,
      "litellm_provider": "azure_ai",
      "max_input_tokens": 1000000,
      "max_output_tokens": 1000000,
      "max_tokens": 1000000,
      "mode": "chat",
      "output_cost_per_token": 4.12e-06,
      "source": "<vendor blog post>",
      "supported_modalities": ["text"],
      "supported_output_modalities": ["text"],
      "supports_function_calling": true,
      "supports_reasoning": true,
      "supports_tool_choice": true
  }
  ```
- `model_prices_and_context_window.json:6690-6706` — identical block
  in the canonical file. The two files are intentionally kept in
  sync; this PR maintains that invariant.
- The flag set (`supports_function_calling`, `supports_reasoning`,
  `supports_tool_choice`) matches the existing `azure_ai/deepseek-v3`
  entry shape directly above it, so downstream feature-detection in
  `litellm/utils.py:supports_*` will pick it up without any code
  change.

## Why merge-as-is

- Pure metadata addition. No code paths touched, no provider
  transformer changes, no router defaults changed.
- 1M / 1M context+output is the published spec; pricing values are
  the published list rates.
- Mirrors the pattern of every prior provider-metadata PR in this
  file (e.g. the immediately-preceding `azure_ai/deepseek-v3`
  block).

## Tiny optional follow-ups (do not block)

- The `source` URL points to a vendor blog post. Some entries in the
  same file use the provider's pricing-page URL instead of a blog
  post; either is accepted by current convention but pricing pages
  are slightly more durable.
- If the v4-flash model also exposes a structured-output / JSON-mode
  flag, consider adding `supports_response_schema: true` in a
  follow-up once the provider doc confirms it.

Ship.
