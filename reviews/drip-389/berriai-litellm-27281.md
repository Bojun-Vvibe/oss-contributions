# BerriAI/litellm#27281 — Fix LLM judge guardrail execution in Playground

- Head SHA: `0666795e62cc2aaebedced632c98123bdbb4de1c`
- Author: @oss-agent-shin
- Link: https://github.com/BerriAI/litellm/pull/27281

## Notes
- `litellm/proxy/guardrails/guardrail_hooks/llm_as_a_judge/__init__.py:97` adds `llm_router: Optional["Router"] = None` to `__init__`, plus `self.buffer_streaming_response = True` at line 132 — required so the judge can score the full response. Sensible default for a judge guardrail.
- `_run_judge` at `__init__.py:147–161` picks `self.llm_router.acompletion` over `litellm.acompletion` when the router is present. Crucially, it injects `metadata={"user_api_key_metadata": {"disable_global_guardrails": True}, "user_api_key_team_metadata": {"disable_global_guardrails": True}}` to prevent recursive guardrail evaluation on the judge call itself — this is the real fix and worth a comment in code.
- `initialize_guardrail` at `__init__.py:261` now threads `llm_router` through; ensure the Playground entry point actually passes the router (otherwise the optional path is dead).

## Verdict
`merge-after-nits`

Correct fix for both the routing and the recursion concerns. Add a code comment on the `disable_global_guardrails` metadata explaining *why* (otherwise a future cleanup will delete it), and add a test that verifies the judge call does not re-trigger the parent guardrail.
