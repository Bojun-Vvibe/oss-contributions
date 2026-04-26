---
pr: 26538
repo: BerriAI/litellm
sha: 88fe6c9ccfb1
verdict: merge-after-nits
date: 2026-04-26
---

# BerriAI/litellm #26538 — fix(fireworks_ai): modernize chat transforms, add Messages + Responses configs

- **URL**: https://github.com/BerriAI/litellm/pull/26538
- **Author**: frdeng
- **Head SHA**: 88fe6c9ccfb1
- **Size**: +1005/-172 across 12 files (`litellm/llms/fireworks_ai/{chat,messages,responses}/transformation.py`, `cost_calculator.py`, `litellm/__init__.py`, `_lazy_imports_registry.py`, `utils.py`, 4 test files)

## Scope

Three logical changes packaged together:

1. **Chat transform modernization**: documents the existing fireworks-specific quirks as docstrings on `FireworksAIConfig` (`chat/transformation.py:39-90`), adds `custom_llm_provider` property, expands `get_supported_openai_params` with `top_logprobs`, `seed`, `logit_bias`, `parallel_tool_calls`, `thinking`, `reasoning_effort`, removes unused imports (`OpenAIChatCompletionToolParam`, `supports_reasoning`).
2. **New `FireworksAIMessagesConfig`** subclassing `AnthropicMessagesConfig` (`messages/transformation.py:494`) — Anthropic-compatible Messages API surface for Fireworks-hosted Anthropic-shaped models.
3. **New `FireworksAIResponsesConfig`** subclassing `OpenAILikeResponsesConfig` (`responses/transformation.py:563`) — OpenAI Responses API surface.

Wiring: both new configs registered in `litellm/__init__.py:1798-1803`, `_lazy_imports_registry.py:1014-1022`, and `utils.py:1134-1140` and `:1339-1346` for provider dispatch.

## Specific findings

- **Docstring documentation pass at `chat/transformation.py:39-90` is excellent** — it's the kind of "what does this provider quirk-handle that base OpenAI doesn't" reference future contributors will actually read. The "Tool-calls-in-content workaround" note explicitly calling out Llama-v3p3-70b vs newer Kimi/MiniMax/GLM/DeepSeek is exactly the kind of vendor-quirk archaeology that disappears from chat logs and becomes mystery code in 18 months.

- **Three logically independent changes in one PR**. The chat docstring/param expansion, the new Messages config, and the new Responses config are independently mergeable. If any one of them has a regression risk that needs rollback, the whole PR has to revert. Maintainer should ask for a split or, at minimum, three commits with conventional prefixes so a partial revert is `git revert <sha>` rather than a manual diff.

- **`reasoning_effort` is now passed through unconditionally** (per the docstring at `chat/transformation.py:88` and `get_supported_openai_params:155`) on the basis that "the Fireworks API accepts it on all models and handles unsupported cases itself". This is a contract change: previously, litellm gated `reasoning_effort` via `supports_reasoning(model=...)` (the deleted import is the smoking gun). The new behavior delegates filtering to upstream. Two risks: (a) Fireworks could change behavior and start 400-ing on unsupported models, leaving litellm callers surprised; (b) cost-tracking code that assumed "if reasoning_effort was sent, the response has reasoning tokens" might now over-count. Worth a one-line note in the changelog and a regression test that asserts a non-reasoning Fireworks model accepts `reasoning_effort: "low"` without error and returns no reasoning tokens.

- **`custom_llm_provider` property addition at `chat/transformation.py:88`** returning `"fireworks_ai"` is the right shape — most provider configs already do this and base `OpenAIGPTConfig` returns `"openai"` by default. Confirms downstream code (cost calc, logging, callback router) sees the provider correctly.

- **`FireworksAIMessagesConfig` inherits from `AnthropicMessagesConfig`** (`messages/transformation.py:494`). That's the right base — Fireworks does host Anthropic-shaped models — but reviewer should confirm: (a) the auth header name matches Fireworks (Anthropic uses `x-api-key`, Fireworks uses `Authorization: Bearer`); (b) the model-name prefixing rule (`accounts/fireworks/models/<model>`) applies in Messages mode too; (c) `cache_control` stripping that the chat path does also fires here, since Fireworks's Anthropic surface may have different cache semantics than Anthropic-direct.

- **`FireworksAIResponsesConfig` inherits from `OpenAILikeResponsesConfig`** (`responses/transformation.py:563`). Inheriting from `OpenAILikeResponsesConfig` rather than `OpenAIResponsesConfig` is the right choice (OpenAI-like = "fork the protocol, change the auth/url/quirks"). Same review questions as Messages: confirm model-prefixing, auth header, and the response-side `tool_calls`-in-content workaround is preserved or deliberately omitted.

- **Lazy-import registration at `_lazy_imports_registry.py:1014-1022`** is correctly placed alphabetically next to `FireworksAIEmbeddingConfig`. Good.

- **Test file split is good**: separate `test_fireworks_ai_chat_transformation.py`, `test_fireworks_ai_cost_calculator.py`. Reviewer should spot-check that the new Messages and Responses configs have *their own* test files (visible in the diff: `test_fireworks_ai_translation.py` exists at minimum). 12 files / 1005 lines means there's room.

- **`utils.py` provider dispatch additions at `:1134` and `:1339`**: these are in the central `get_llm_provider` / `get_optional_params` machinery. Any time a provider gets two new dispatch arms in `utils.py`, the regression risk is "did I shadow another provider's branch". Reviewer should view the surrounding context to confirm the new branches are gated tightly enough on `custom_llm_provider == "fireworks_ai" and route in {"messages", "responses"}`.

- **No deprecation note** for the removed `supports_reasoning(model=..., custom_llm_provider="fireworks_ai")` capability check. If any external library or user code introspected litellm's `supports_reasoning` for Fireworks models to decide whether to send `reasoning_effort`, that contract is now ambiguous. Worth a one-line note that `supports_reasoning` is deliberately no longer consulted for Fireworks and the upstream API handles it.

## Risk

Medium. The chat transform changes are documentation + safe param expansion + a contract-change on `reasoning_effort` filtering. The two new configs are new surfaces (low risk to existing flows) but their inheritance choices need a careful look at auth/url/prefixing parity. Central `utils.py` dispatch additions are the highest-risk region — easy to land a branch that catches more than intended.

## Nits

1. Split into three commits (chat modernization / Messages config / Responses config) for safer partial revert.
2. Add a regression test for `reasoning_effort` on a non-reasoning Fireworks model (round-trip without 400).
3. Confirm Messages and Responses configs preserve model-name prefixing, auth header shape, and `cache_control` stripping.
4. Note in the changelog that `supports_reasoning` is no longer consulted for Fireworks reasoning_effort gating.
5. Spot-check the `utils.py:1134`/`:1339` dispatch arms for tight gating.
6. Add a `parallel_tool_calls` test exercising Fireworks's variant of the OpenAI tool_call shape.

## Verdict

**merge-after-nits** — solid documentation pass and two reasonable new surfaces, but the bundling makes the diff harder to land safely than it needs to be, and the `reasoning_effort` gating change deserves an explicit note. Inheritance choices for the two new configs are right but need confirmation on parity with the chat path's vendor quirks.

## What I learned

When a provider-config class accumulates docstring archaeology like "tool calls returned in content for older Llama-v3p3-70b but not for newer Kimi/MiniMax/GLM/DeepSeek", that's a sign it's time to either (a) add a `model_capabilities` table that drives the workaround conditionally or (b) wait for the legacy models to age out and delete the workaround. Documenting it as the docstring does is the right interim step — the next contributor at least knows *why* the special case exists and which models it covers, and the docstring is greppable when a user files a bug like "tool calls broken on `accounts/fireworks/models/llama-v3p3-70b-instruct`".
