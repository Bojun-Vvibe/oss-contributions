# BerriAI/litellm #27080 — fix(agentic): preserve custom api_base/api_key in agentic follow-up requests

- **PR:** https://github.com/BerriAI/litellm/pull/27080
- **Head SHA:** `e6cdcb0188e65d421213949aee12f9761b8ec170`
- **Author:** jesset
- **Size:** +458 / -4 across 7 files
- **Fixes:** #26389, #26323

## Summary

When the agentic websearch loop made follow-up requests, it was dropping the user-supplied `api_key` / `api_base` and falling back to `api.anthropic.com`. Same bug surfaced in `count_tokens` for non-Anthropic Anthropic-compatible endpoints (Tencent's `lkeap.cloud.tencent.com/plan/anthropic`, etc.). Fix threads the originally-supplied creds through both code paths.

## Specific references

- `litellm/integrations/websearch_interception/handler.py:756-787` — pulls `api_key` / `api_base` from `logging_obj.model_call_details["agentic_loop_params"]` and threads them into the follow-up request. Critically, the patch *removes* matching keys from `request_patch.kwargs` before splatting (`followup_kwargs.pop("api_key", None)` / `pop("api_base", None)`), avoiding the duplicate-keyword TypeError that would otherwise hit when the patch already supplied them. That's the right ordering.
- `litellm/llms/anthropic/count_tokens/token_counter.py:55-75` — derives `count_tokens_api_base` from the deployment's `api_base` by appending `/v1/messages/count_tokens`. Comment shows the concrete Tencent example (`https://api.lkeap.cloud.tencent.com/plan/anthropic` → `…/v1/messages/count_tokens`). The `rstrip("/")` before concat correctly handles trailing-slash bases.
- `litellm/llms/anthropic/experimental_pass_through/messages/handler.py` — parallel fix in the experimental pass-through path.
- `litellm/llms/custom_httpx/llm_http_handler.py` — supporting plumbing.
- Test coverage:
  - `tests/test_litellm/integrations/websearch_interception/test_websearch_thinking_constraint.py`
  - `tests/test_litellm/llms/anthropic/count_tokens/test_count_tokens_api_base.py`
  - `tests/test_litellm/llms/custom_httpx/test_llm_http_handler.py`

## Concerns

- The conditional `**({"api_key": _followup_api_key} if _followup_api_key else {})` pattern is repeated twice. Functionally fine; a tiny helper or just `kwargs = {**followup_kwargs}; if k: kwargs["api_key"] = k` would read cleaner. Stylistic only.
- `count_tokens_api_base` assumes the deployment's `api_base` does not already include `/v1/messages` or similar suffix. For the Tencent example shown that's true, but a defensive check (`if "/messages" in api_base: …` warn) would be friendlier to operators with unusual base URLs.
- The fix correctly handles `api_base=None` by leaving `count_tokens_api_base=None` and falling back to existing logic — good.

## Verdict

**merge-after-nits** — addresses a real silent-fallback bug that would route private agentic traffic to the wrong endpoint. Nits: defensive log/warn if the supplied `api_base` looks like it already ends in a messages path, plus optional readability cleanup of the splat pattern.
