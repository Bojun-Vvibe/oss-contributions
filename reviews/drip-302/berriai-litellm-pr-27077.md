# BerriAI/litellm PR #27077 — test(responses): replace legacy `claude-4-sonnet-20250514` alias in multiturn tool-call test

- Head SHA: `2e6965381e68`
- Files changed:
  - `tests/llm_responses_api_testing/test_anthropic_responses_api.py` (2 line replacements)

## Analysis

Pure test maintenance. The test `test_multiturn_tool_calls` was hard-coded against Anthropic alias `claude-4-sonnet-20250514`, which now returns:

```
litellm.NotFoundError: AnthropicException - {"type":"error","error":{"type":"not_found_error","message":"model: claude-4-sonnet-20250514"}}
```

PR replaces it with `anthropic/claude-haiku-4-5-20251001` at two call sites:

- `tests/llm_responses_api_testing/test_anthropic_responses_api.py:92` — initial `litellm.responses(...)` call.
- `tests/llm_responses_api_testing/test_anthropic_responses_api.py:118` — follow-up `litellm.responses(...)` call sharing `previous_response_id=response_id`.

Both lines now use the same model string, which is correct: a multiturn responses-API session must thread the same model across follow-up calls or you risk testing cross-model behavior unintentionally.

Why haiku-4-5 is fine here:
- The test exercises the `tool_calls` shape end-to-end (shell tool, follow-up message). It does not assert anything model-specific; any current Anthropic model that supports tools and the `/responses` shape will exercise the same client code path.
- Haiku is cheaper than Sonnet, which is a small win for CI cost on `llm_responses_api_testing/`.

Risks / nits:
- Anthropic model strings are still hard-coded throughout this test file. If the goal is to keep CI from breaking on alias retirements, the durable fix is a `LATEST_ANTHROPIC_TOOL_MODEL` constant at the top of the file. Out of scope for a 1-line fix, but worth noting as a follow-up.
- No fallback / `pytest.skip` if `ANTHROPIC_API_KEY` is unset, but that already matches the rest of the file's pattern.
- PR was clearly authored by an automated agent (`<!-- CURSOR_AGENT_PR_BODY_BEGIN -->` marker in the body). Diff is minimal and human-verifiable, so the agent provenance is not a blocker.

## Verdict

`merge-as-is`

## Nits

- Follow-up: hoist the model string into a module-level constant so the next alias retirement is a one-line change in one place.
