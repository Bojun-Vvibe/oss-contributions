# Review: BerriAI/litellm #27283 — feat(openai-like): add neosantara provider

- Carrier: `BerriAI/litellm`
- PR: #27283
- Head SHA: `973120a3`
- Drip: 387

## Summary
Verdict: `merge-as-is` (mas)

Standard JSON-driven OpenAI-like provider registration for Neosantara
(`https://api.neosantara.xyz/v1`). Four touch points, each minimal and in
the established pattern:
1. README provider table row (alphabetically inserted between `nebius` and
   `nlp_cloud`).
2. New entry in `litellm/llms/openai_like/providers.json` with
   `base_url` / `api_key_env` / `api_base_env` plus an explicit
   `supported_endpoints: ["/v1/chat/completions", "/v1/responses"]`.
3. New `LlmProviders.NEOSANTARA = "neosantara"` enum value in
   `litellm/types/utils.py`.
4. A 5-test pytest fixture exercising registration, env-var resolution,
   URL construction, `get_llm_provider` resolution, and
   `ProviderConfigManager.get_provider_chat_config`.

## Diff anchors
- `README.md:333` — provider table row, correctly preserves column
  alignment (10 status columns, all `✅ ✅ ✅` for the supported triad
  and blanks for the rest).
- `litellm/llms/openai_like/providers.json:109-118` — the new entry.
  Notably, this is the first OpenAI-like provider in the JSON registry
  to declare `supported_endpoints` — every other entry omits the field.
  The dynamic config loader (`dynamic_config.py` / `json_loader.py`)
  must already tolerate this field for the test to pass.
- `litellm/types/utils.py:3331` — enum value insertion in the existing
  alphabetical-by-recent-add block (after `XIAOMI_MIMO`, before
  `LITELLM_AGENT`). Consistent with the file's append-only convention.
- `tests/litellm/llms/openai_like/test_neosantara_provider.py:1-81` —
  new test module. Five tests:
  - `test_neosantara_provider_registered` asserts the JSON registry
    entry round-trips (`base_url`, env names, `/v1/responses` in
    `supported_endpoints`).
  - `test_neosantara_resolves_env_api_key` uses `monkeypatch.setenv` to
    verify env-var resolution via `_get_openai_compatible_provider_info`.
  - `test_neosantara_complete_url_appends_endpoint` asserts the
    canonical `<base>/chat/completions` URL shape.
  - `test_neosantara_provider_resolution` exercises the full
    `get_llm_provider` parser entry point with `model="neosantara/
    claude-4.5-sonnet"`, asserting the model strip + provider tag +
    api_base default + `api_key=None` (since env not set).
  - `test_neosantara_provider_config_manager` checks
    `ProviderConfigManager.get_provider_chat_config` returns a config
    whose `custom_llm_provider == "neosantara"`.

## Concerns / nits
1. The `claude-4.5-sonnet` model name used in tests is a placeholder
   (Anthropic doesn't publish that exact tag); if Neosantara doesn't
   actually expose that name, the tests still pass (they only exercise
   the parser and config-manager wiring, not a real network call) but
   the choice is misleading. A more obviously-fake `test-model` would
   document intent better.
2. No model-list entry in `model_prices_and_context_window_backup.json`
   (or the corresponding upstream source). This means cost tracking,
   token-limit awareness, and model-routing decisions for
   `neosantara/*` models all silently default to whatever the
   `openai_like` fallback dictates. Acceptable for an initial provider
   onboarding (the maintainer pattern is to add models in a follow-up
   PR once the user reports them), but worth a CHANGELOG note.
3. The `supported_endpoints` array is the first of its kind in
   `providers.json`; if a maintainer hasn't reviewed how
   `dynamic_config` consumes this field for other code paths
   (routing, capability detection, `/v1/models` listing), there's a
   nonzero chance of unintended downstream coupling.
4. Test file lives at `tests/litellm/llms/openai_like/` not
   `tests/test_litellm/llms/openai_like/` — a quick `rg
   "litellm/llms/openai_like" tests/` confirms which convention this
   repo currently uses.

## Risk
Low. Pure provider-registration patch; no change to any shared code path.
The five tests cover the wiring points that matter. Worst case: a user
hits an unknown-model error if their model name doesn't match what
Neosantara actually exposes, which is the same failure mode every
new-provider PR has on day one.
