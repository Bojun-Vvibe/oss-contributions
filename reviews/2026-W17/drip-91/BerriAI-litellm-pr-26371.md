---
pr: 26371
repo: BerriAI/litellm
sha: 7fbd182670accf853765070cf647a402fce12182
verdict: merge-as-is
date: 2026-04-27
---

# BerriAI/litellm #26371 — feat: add Telnyx as OpenAI-compatible provider

- **Head SHA**: `7fbd182670accf853765070cf647a402fce12182`
- **URL**: https://github.com/BerriAI/litellm/pull/26371
- **Size**: small (268/0 across 6 files; ~140 lines are tests)

## Summary
Registers Telnyx (`https://api.telnyx.com/v2/ai`, `TELNYX_API_KEY`) as a new OpenAI-compatible provider via the JSON provider registry. Pure additive: docs page, sidebar entry, providers.json registration, enum value, and an endpoints-support matrix entry, plus a dedicated test file.

## Specific findings
- `litellm/llms/openai_like/providers.json:104-108` — minimal correct registration: `{"base_url": "https://api.telnyx.com/v2/ai", "api_key_env": "TELNYX_API_KEY"}`. No `param_mappings` block — appropriate, since the docs page shows Telnyx accepts the OpenAI-native parameter names directly.
- `litellm/types/utils.py:3308` — `TELNYX = "telnyx"` added to `LlmProviders` enum. Lower-case enum value matches the `providers.json` key, which matches the URL prefix the user types in `model="telnyx/..."`. This three-way agreement is what the JSON provider system relies on.
- `provider_endpoints_support.json:2014-2030` — the endpoints matrix declares `chat_completions: true, embeddings: true` and the rest `false`. This matches the documented Telnyx surface (their docs page lists chat completions and an embeddings example, not Responses/Messages/audio/image). One small pin: the value of `messages: false` is technically conservative since Telnyx is OpenAI-compatible and could route via the `/v1/messages` shim, but conservative is the right call until tested.
- `tests/test_litellm/llms/openai_like/test_telnyx_provider.py:18-78` — four solid tests:
  - `test_telnyx_provider_exists` — asserts registry presence (catches typos in `providers.json` key).
  - `test_telnyx_provider_config` — asserts the URL and env var.
  - `test_telnyx_dynamic_config_generation` — asserts `create_config_class()` round-trips and verifies the `_get_openai_compatible_provider_info()` override path (custom base / key).
  - `test_telnyx_provider_resolution` — asserts `get_llm_provider("telnyx/meta-llama/...")` correctly splits provider and model.
- `docs/my-website/docs/providers/telnyx.md` — full docs page including quick-start, OpenAI SDK alternative, available models table, embeddings, proxy config, and signup link. The model table at lines 73-77 lists three placeholders (`Kimi-K2.6`, `GLM-5.1-FP8`, `MiniMax-M2.7`) that look like internal codenames — verify these are the names Telnyx actually exposes today, otherwise users will hit "model not found" on first try. The link to `developers.telnyx.com/docs/inference/models` is the right escape hatch.
- `docs/my-website/sidebars.js:1012` — alphabetically inserted between `synthetic` and `snowflake`. Note: that ordering is not actually alphabetical — `snowflake` should come before `synthetic`. Pre-existing inconsistency, not this PR's fault, but the new `telnyx` entry inherits the disorder. Worth a one-line fix in a follow-up.

## Risk
Low. JSON-registry providers are isolated by construction; a misconfigured registration only breaks `model="telnyx/..."` requests and doesn't touch other providers. The test file covers the four failure modes I'd care about (registration, config, dynamic class, resolution).

## Verdict
**merge-as-is** — clean additive provider registration, full test coverage of the JSON-registry plumbing, conservative endpoints-support matrix. Verify the model names in the docs table against Telnyx's live model list before final merge if the maintainer hasn't already.
