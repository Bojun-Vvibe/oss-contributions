# BerriAI/litellm#26639 — preserve dynamic api_base/api_key in websearch agentic follow-up

- **PR**: https://github.com/BerriAI/litellm/pull/26639
- **Author**: @skgandikota
- **State**: OPEN
- **Scope**: medium — 5 files (+236/-2). 2 source files, 1 typed-dict, 2 mocked unit-test files.

## Summary

Fixes a real cooldown-cascade bug (issue #26389): when `websearch_interception` runs the agentic loop for an Anthropic-shaped model whose deployment uses a custom `api_base` / `api_key`, the *follow-up* request after the search-results turn was being authenticated with the env-var key against the provider's default endpoint (`https://api.anthropic.com`) instead of the deployment's configured endpoint and key. Result: `401 Unauthorized` → deployment marked unhealthy → cooldown → cascade as adjacent traffic falls back. Two contributing locations are fixed:

1. The Anthropic experimental pass-through handler stored only `model` + `custom_llm_provider` from `litellm.get_llm_provider(...)`'s 4-tuple return into `agentic_loop_params`, dropping the `dynamic_api_key` and `dynamic_api_base` siblings.
2. The websearch interception handler's `_build_anthropic_request_patch` built `kwargs_for_followup` without re-injecting `api_key` / `api_base`, so the follow-up `anthropic_messages.acreate(...)` had no path to recover deployment credentials.

## Diff anchors

- `litellm/llms/anthropic/experimental_pass_through/messages/handler.py` (`agentic_loop_params` block) — now persists `api_key` / `api_base` only when `dynamic_*` are not `None`. The `if not None` gate is the right shape: the default provider-config code path (where `get_llm_provider` returns `None` for both dynamic fields) is byte-identical to before, so nothing that was working keeps working — only the previously-broken deployment-override path changes.
- `litellm/integrations/websearch_interception/handler.py` (`_build_anthropic_request_patch`) — propagates the credentials into `kwargs_for_followup` with the `"api_key" not in kwargs_for_followup` / `"api_base" not in kwargs_for_followup` guard. Caller-supplied values win over the captured `agentic_loop_params` copies. This precedence is correct: explicit caller intent at the followup site beats the captured credentials from the original request.
- `litellm/types/utils.py` — `AgenticLoopParams` TypedDict gets optional `api_key` / `api_base` fields with docstrings. Right shape — keeps the type definition in sync with the runtime payload, so type-checkers won't silently strip the new fields downstream.
- `tests/test_litellm/llms/anthropic/.../test_anthropic_experimental_pass_through_messages_handler.py` — two new tests: `test_agentic_loop_params_preserves_dynamic_api_key_and_api_base` (positive: when `get_llm_provider` returns `("…", "…", "deployment-key", "https://my-proxy.example.com/v1")`, both fields land in `agentic_loop_params`) and `test_agentic_loop_params_omits_keys_when_dynamic_values_are_none` (negative: when both are `None`, neither key is *present* — not just `None`-valued — in the dict, preserving the existing fallback resolution logic). The "omit, don't None" assertion is the load-bearing one — without it, downstream `agentic_params.get("api_key")` would return `None` instead of falling through to the next resolution layer.
- `tests/test_litellm/integrations/websearch_interception/test_websearch_interception_handler.py` — two new tests: `test_followup_request_preserves_custom_api_base_and_api_key` (seeds `agentic_loop_params`, asserts `plan.request_patch.kwargs` carries the credentials through) and `test_followup_request_does_not_override_user_provided_api_credentials` (precedence: explicit kwargs win). This second test pins the precedence contract; without it, a future "always overwrite" refactor could regress silently.
- Test-suite signal: `131 passed, 2 skipped, 2 warnings in 22.74s` for the two affected modules. Acceptable.

## What I'd push back on

1. **No test for the cooldown-cascade root cause.** All four new tests are pure unit tests on the credential propagation path; none assert "the followup actually goes to the configured `api_base` and not the default." A mocked `httpx`/`AsyncClient` integration test that captures the request URL and `Authorization` header on the followup call would pin the actual user-visible bug fix, not just the data-flow plumbing.
2. **No regression test for streaming followups.** The agentic loop has both streaming and non-streaming code paths through `anthropic_messages.acreate`. The new tests exercise non-streaming only.
3. **No coverage of the OpenAI / Bedrock / Vertex variants** of the websearch interception path. If those backends have parallel `_build_*_request_patch` implementations with the same bug, this PR fixes only the Anthropic one. Worth a grep + a follow-up issue at minimum.
4. **`agentic_params.get("api_key")` returns the raw key string** — make sure log redaction in the followup request path still masks it. The original deployment-credential masking lives at the litellm-router layer; bypassing the router on followup may bypass redaction. Add an assertion in the integration test.

## Verdict

**merge-after-nits** — correct fix shape (closes a real 401-cascade with the right precedence semantics, type def kept in sync, four tests pin both presence and absence cases), but add at least one integration-style test that asserts the followup request actually targets the custom `api_base` with the custom `api_key`. The current tests prove the data flows; they don't prove the bug is fixed end-to-end.

Repo coverage: BerriAI/litellm (websearch_interception agentic-loop credential propagation across Anthropic experimental pass-through followup requests).
