# Review — BerriAI/litellm#27088 — feat: WebFetch interception with Firecrawl support

- PR: https://github.com/BerriAI/litellm/pull/27088
- Head: `3abab76fe503c4c1a44f06c22097d3d396eb4dff`
- Verdict: **needs-discussion**

## What the change does

Adds a sizeable new subsystem under
`litellm/integrations/webfetch_interception/` with `__init__.py`,
`handler.py` (~1043 lines), `tools.py`, and `transformation.py`. Defines
a new constant `LITELLM_WEB_FETCH_TOOL_NAME = "litellm_web_fetch"` in
`litellm/constants.py:516`. The `WebFetchInterceptionLogger`
(`CustomLogger`) intercepts WebFetch tool calls for providers (default
`[LlmProviders.BEDROCK]`) that don't natively support server-side fetch,
and runs an agentic loop: detect → execute fetch via the router → make a
follow-up call → return the synthesized response.

## Concerns

1. **Surface area.** A 1043-line `handler.py` introducing a new agentic
   loop, a new transformation layer, and new tool conventions in one PR
   is hard to review safely. Suggest splitting:
   (a) introduce the new tool name + `tools.py` helpers + tests,
   (b) introduce `transformation.py` with focused unit tests,
   (c) introduce the agentic loop handler last, gated behind config.
2. **Default `enabled_providers = [LlmProviders.BEDROCK.value]`** when
   `enabled_providers is None`. The docstring above it says "If None or
   empty list, enables for ALL providers" — the code disagrees. Pick one
   semantic and align both. Recommend default = `[]` meaning disabled,
   so a fresh install is opt-in.
3. **`async_pre_call_deployment_hook`** mutates incoming `kwargs["tools"]`
   to convert native `web_fetch` tools to "regular" ones. Confirm this
   doesn't leak across requests if `kwargs["tools"]` is a shared
   reference (common in router caching paths). Defensive `copy.deepcopy`
   on the tools list before mutation is cheap insurance.
4. **`is_web_fetch_tool`** is asked to recognize "native or already
   LiteLLM standard"; the matching rules deserve explicit unit tests for
   each provider's tool shape (Anthropic native vs OpenAI-style
   function-tool wrapper) — those tests aren't visible in this diff.
5. **Provider detection fallback** silently swallows exceptions from
   `litellm.get_llm_provider` and treats the result as "not enabled" —
   that's safe, but please log at debug level so misconfigured custom
   providers are diagnosable.
6. No new test files appear in the truncated diff sample. A feature this
   size needs golden tests for at least Bedrock + one OpenAI-compatible
   path before merge.

Hold for split + opt-in default + tests.
