# BerriAI/litellm #27080 — fix(agentic): preserve custom api_base/api_key in agentic follow-up requests

- PR: https://github.com/BerriAI/litellm/pull/27080
- Author: jesset
- Head SHA: `ae337448936c65e737e1bf845acd8da71d0e1fa0`
- Updated: 2026-05-03T10:20:07Z

## Summary
Fixes a real bug for users routing Anthropic-shaped traffic through third-party providers (Tencent Cloud, Bedrock, etc.): when the websearch interception agentic loop made follow-up requests, it dropped the original `api_key`/`api_base` and fell back to `api.anthropic.com` + `ANTHROPIC_API_KEY`. Three coordinated changes: (1) capture `dynamic_api_key`/`dynamic_api_base` into `agentic_loop_params` at the entry point (`litellm/llms/anthropic/experimental_pass_through/messages/handler.py:368-385`); (2) propagate them in the websearch interception handler's follow-up `acreate` call (`litellm/integrations/websearch_interception/handler.py:756-784`); (3) same propagation in the HTTP handler's agentic plan executor (`litellm/llms/custom_httpx/llm_http_handler.py:4642-4690`). Also a small unrelated-but-adjacent fix in `count_tokens/token_counter.py` to derive a `count_tokens_api_base` from `api_base`. New tests in `tests/test_litellm/integrations/websearch_interception/test_websearch_thinking_constraint.py` (`TestAgenticLoopCredentialsForwarding`).

## Observations
- `litellm/llms/anthropic/experimental_pass_through/messages/handler.py:371-381`: stores `api_key`/`api_base` only when `dynamic_api_key`/`dynamic_api_base` are not None. Good — avoids overwriting downstream consumers that explicitly want None. The dict literal was changed to a build-up-then-assign pattern, which is the right shape.
- `litellm/integrations/websearch_interception/handler.py:758-775`: the `followup_kwargs.pop("api_key", None); followup_kwargs["api_key"] = ...` pattern is defensive against `request_patch.kwargs` already having those keys. The comment "Avoid duplicate keyword arguments" is good. **Subtle precedence question:** the PR makes `agentic_loop_params` *override* whatever `request_patch.kwargs` has. That's the correct precedence for this bug (user's original creds beat any patch-time defaults), but worth a comment explaining the choice — future readers will assume patch.kwargs is more specific.
- `litellm/llms/custom_httpx/llm_http_handler.py:4682-4688`: same pattern, same precedence. Consistent across both call sites — good.
- `litellm/llms/anthropic/count_tokens/token_counter.py:55-78`: the `count_tokens_api_base = f"{base}/v1/messages/count_tokens"` construction assumes the api_base is the Anthropic root, not already a path-suffixed endpoint. The example in the comment (`https://api.lkeap.cloud.tencent.com/plan/anthropic`) clarifies. **Potential bug:** if a user already configured `api_base = "https://x/v1/messages"`, this would produce `.../v1/messages/v1/messages/count_tokens`. Not a regression (previously `count_tokens` would 404 entirely for those users), but worth either a regex check or a comment that api_base must be the provider root.
- `tests/.../test_websearch_thinking_constraint.py:48-62`: the new `_make_logging_obj_with_credentials` helper wires `api_key="sk-test-agentic-key"` and `api_base="https://custom.provider.host/plan/anthropic"` into `agentic_loop_params`. The new `TestAgenticLoopCredentialsForwarding` class (line ~493+) asserts both values land in `captured_kwargs` after `_execute_agentic_loop`. Correct boundary — the test would have failed against the old code.
- Missing test: a parallel test for `_execute_anthropic_agentic_plan` in `llm_http_handler.py:4642`. The websearch interception path is covered, but the HTTP-handler agentic plan path is not. Both have the same fix; both deserve a test.
- Nit: the new `Optional[str]` typing on `_followup_api_key`/`_followup_api_base` in `handler.py:758-759` should match the existing typing style in the file. Looks fine but worth a once-over.

## Verdict
`merge-after-nits`
