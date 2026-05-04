# BerriAI/litellm #26971 — feat: add Cortecs AI as named OpenAI-compatible provider

- SHA: `19da4686f24ef84ef2947644884f596f90df18b7`
- State: OPEN, +38/-0 across 3 files

## Summary

Registers `cortecs` as an OpenAI-compatible provider: adds the base URL/api-key env entry to `providers.json`, and declares `chat_completions` support in both copies of `provider_endpoints_support.json`. Pure registry/config addition, no code paths touched.

## Notes

- `litellm/llms/openai_like/providers.json:109-112` — entry includes `base_url` and `api_key_env` but omits `api_base_env`, unlike the immediately preceding `aihubmix` entry which declares `AIHUBMIX_API_BASE`. If users need to point Cortecs at a regional/staging endpoint, they'd have to override at call time. Worth adding `"api_base_env": "CORTECS_API_BASE"` for consistency with the rest of the registry — it's a no-cost safety net.
- `provider_endpoints_support.json:512-528` and `litellm/provider_endpoints_support_backup.json:477-493` — the duplicate-file pattern is followed correctly, and both stanzas are byte-identical (verified by inspection: same `display_name`, same `url` doc link, same endpoint capability matrix). Authors should confirm in the PR description that maintaining both copies in sync is the canonical workflow vs. a generated file — no Makefile target or CI check is referenced.
- `provider_endpoints_support.json:514` — `"url": "https://docs.litellm.ai/docs/providers/cortecs"` references a docs page that the PR does not create. Either add a stub `docs/my-website/docs/providers/cortecs.md` (the convention for every other provider in this file) or change the URL to Cortecs' own docs until the LiteLLM docs page exists, otherwise the UI will surface a 404.
- All capability flags except `chat_completions` are `false`. That matches Cortecs' OpenAI-compatible surface today, but if Cortecs supports `embeddings` or `responses` (most aggregator providers do) it's worth a quick check against their actual API — over-conservatism here just hides functionality from users.
- No tests, no model-cost entries in `model_prices_and_context_window.json`. Pure provider registration; standard for this kind of PR.

## Verdict

`merge-after-nits` — the change is mechanically correct and follows the established 3-file pattern. Add `api_base_env` for parity with neighboring providers, ship a docs stub (or fix the doc URL), and double-check the endpoint capability matrix against Cortecs' actual API surface. Then merge.
