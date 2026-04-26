# Aider-AI/aider PR #5019 — feat: add Kyma API models (via OpenAI-compatible endpoint)

- **PR:** https://github.com/Aider-AI/aider/pull/5019
- **Author:** sonpiaz
- **Head SHA:** `b18cc55e8334`
- **Stats:** +155 / -1 across 2 files (`aider/resources/model-metadata.json` + `aider/resources/model-settings.yml`)
- **Verdict:** `request-changes`

## What this changes

Adds eight model entries for the third-party "Kyma API" gateway (`https://kymaapi.com/v1`, OpenAI-compatible) to aider's resource bundles. Each model gets a metadata block in `model-metadata.json:712-803` with token limits and per-token pricing, and a corresponding settings block in `model-settings.yml:2960-3013` with `edit_format: diff`, `use_repo_map: true`, `examples_as_sys_msg: true`, and `extra_params.max_tokens: 8192`. The eight names are `openai/qwen-3.6-plus`, `openai/deepseek-v3`, `openai/deepseek-r1`, `openai/kimi-k2.5`, `openai/gemma-4-31b`, `openai/qwen-3-32b`, `openai/llama-3.3-70b`, and `openai/minimax-m2.5`.

## Why request-changes

**Half of these model names don't appear to correspond to real upstream releases.** `qwen-3.6-plus` and `qwen-3-32b` aren't in QwenLM's published lineup (the actual 32B-parameter Qwen3 release is `Qwen3-32B`, not `qwen-3-32b`, and there's no `3.6-plus` variant — the 2026 lineup is Qwen3.5 / Qwen3-Next). `kimi-k2.5` isn't a Moonshot release name (the actual line is `kimi-k2-instruct` / `kimi-k1.5`). `gemma-4-31b` doesn't match Google's Gemma lineup (Gemma 3 caps at 27B; there's no Gemma 4 generally available). `llama-3.3-70b` was a real Meta release, that one's plausible. `deepseek-v3` and `deepseek-r1` are real upstream models. `minimax-m2.5` doesn't match MiniMax's actual `MiniMax-M1`/`MiniMax-Text-01` naming. This pattern — a third-party gateway re-exporting upstream models under arbitrary version-bumped aliases — is exactly the shape of either a vendor's marketing rebrand or a low-quality aggregator that picks names without coordination upstream. Either way, adding these to aider's *first-party* `model-metadata.json` (which ships with every install) will leave a lot of users typing `aider --model openai/kimi-k2.5` and getting an HTTP 404 from whatever endpoint they configured.

**The endpoint is documented only via a comment in `model-settings.yml:2960`.** That comment (`# Kyma API models (OpenAI-compatible gateway, set --openai-api-base https://kymaapi.com/v1)`) is the *entire* documentation of what these models are or where to point at them. There's no provider docs page added under `aider/website/docs/llms/`, no entry in the `OPENAI` provider table, no mention in the README — meaning a user who encounters one of these names in `aider --models` and tries to use it will get a confusing "no API key" error from LiteLLM with no breadcrumb back to "you need to configure a third-party gateway URL." Compare with how `together_ai/` or `openrouter/` models are surfaced: each ships with a docs page explaining the provider and the auth shape.

**The `litellm_provider: "openai"` field is misleading and will collide with real OpenAI namespacing.** Every model is registered as `openai/<name>` with `litellm_provider: openai`, which means LiteLLM's cost-tracking, rate-limiting, and routing will treat these as OpenAI-direct calls. If OpenAI ever ships a model called `gemma-4-31b` or `qwen-3-32b` (unlikely but the namespace is theirs), aider's resource files would silently shadow the real one. The standard pattern for OpenAI-compatible third-party gateways is a separate provider prefix (`kyma/qwen-3.6-plus`) wired via a custom `litellm_provider`, not `openai/<vendor-named-model>`.

**Pricing precision is suspicious.** `input_cost_per_token: 0.000000439` and `0.000000675` and `0.000001188` — these aren't round-number prices that a vendor publishes, they look like exchange-rate conversions from CNY/USD or post-discount aggregator markups. If they're approximations the doc should say so, otherwise aider's `--cost` output will quote false-precision numbers to users.

**`supports_function_calling: false` on `openai/deepseek-r1` is technically right for upstream R1 (which is reasoning-only) but the matching `model-settings.yml:2978-2984` block doesn't add `extra_params.reasoning_effort` or any other reasoning-related setting**, so aider users won't get the reasoning-mode UX even though they're paying reasoning-tier prices.

## What would unblock

1. Move these to a separate `openai/kyma/<model>` namespace or split into a `kyma/<model>` provider block, so they don't shadow real OpenAI names.
2. Add a `aider/website/docs/llms/kyma.md` walkthrough with the `--openai-api-base` and `OPENAI_API_KEY` setup, model availability table, and pricing-source link.
3. Cross-check each model name against the upstream provider's published catalog and either drop the speculative ones or rename them to match the gateway's actual exposed names.
4. If pricing is in CNY-denominated units converted to USD, document the conversion date and rate; if it's the gateway's discounted resale price, document that.

## Bottom line

The mechanical pattern (OpenAI-compatible endpoint exposed via `model-metadata.json` + `model-settings.yml`) is fine, but the specific entries mix real and speculative model names, register them under the wrong provider namespace, and ship without any user-facing docs. Needs a rework before it's safe to expose these to every aider user.
