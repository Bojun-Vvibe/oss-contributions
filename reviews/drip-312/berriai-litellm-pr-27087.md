# BerriAI/litellm PR #27087 — feat: WebFetch interception with Firecrawl support

- URL: https://github.com/BerriAI/litellm/pull/27087
- Head SHA: `df04a955ea553d7e023415aaf07f41314ae9cbd0`
- Verdict: **needs-discussion**

## Summary

Introduces a substantial new subsystem under
`litellm/integrations/webfetch_interception/` that intercepts model
requests carrying a `web_fetch`-shaped tool, converts them to a
LiteLLM-standard `litellm_web_fetch` tool, executes the fetch
server-side (intended for providers like Bedrock/Claude that don't
natively run web_fetch), and feeds results back into an agentic
follow-up call. Adds the constant `LITELLM_WEB_FETCH_TOOL_NAME` and
new modules: `__init__.py`, `handler.py` (~1000 lines), `tools.py`,
`transformation.py`.

## Specific references

- `litellm/constants.py:516` — adds
  `LITELLM_WEB_FETCH_TOOL_NAME = "litellm_web_fetch"`. Mirrors the
  existing `LITELLM_WEB_SEARCH_TOOL_NAME` pattern. Fine.
- `litellm/integrations/webfetch_interception/handler.py:1-70` — module
  docstring claims "agentic loop: detect WebFetch tool_use → execute
  fetch via router's fetch tools → make follow-up request → return
  final response". This is a real new control-flow surface inside the
  proxy. Worth reading carefully.
- `litellm/integrations/webfetch_interception/handler.py` (the
  `async_pre_call_deployment_hook` method, lines ~75-145) — silently
  rewrites `kwargs["tools"]`, replacing any tool that
  `is_web_fetch_tool(t)` matches with `get_litellm_web_fetch_tool_openai()`.
  Also has the side effect:

  ```python
  if kwargs.get("stream"):
      kwargs["stream"] = False
      kwargs["_webfetch_interception_converted_stream"] = True
  ```

  This is a **silent stream→non-stream coercion**. Clients that asked
  for streaming will instead get a buffered response. That's a
  significant API contract change and needs to be either documented
  loudly, opt-in only, or surfaced to the caller.

## Commentary

The motivation (let Bedrock/Claude users use a uniform `web_fetch` tool
shape and have LiteLLM run the actual fetch server-side via Firecrawl)
is reasonable and aligns with the existing web-search interception. The
shape of the code (CustomLogger subclass, `from_config_yaml`
factory, integration via `litellm_settings.webfetch_interception_params`)
follows established patterns.

However, this PR has several characteristics that make me want
discussion before approval, not a "request changes" or "merge":

1. **Silent stream coercion (handler.py).** `kwargs["stream"] = False`
   inside a pre-call hook is the kind of change that breaks production
   integrations in subtle ways: consumers see no streaming output,
   timeouts behave differently, intermediate tokens disappear. At
   minimum this needs to be (a) opt-in via a config flag, (b) emit a
   `verbose_logger.warning`, and (c) ideally raise a clear error so
   callers can decide whether to retry without web_fetch tools.

2. **`enabled_providers` defaults to `[bedrock]` only.** The docstring
   says "If None or empty list, enables for ALL providers" but the
   constructor sets `self.enabled_providers = [LlmProviders.BEDROCK.value]`
   when `enabled_providers is None`. The behavior and the docstring
   contradict each other. Worth pinning down which is intended before
   merge.

3. **~1000 LoC handler with no diff-visible tests.** A new agentic
   loop subsystem that mutates request bodies, suppresses streaming,
   and re-enters the LiteLLM call path needs unit tests for at least
   (a) the tool-shape conversion, (b) the follow-up request assembly,
   (c) error paths when fetch fails, (d) the streaming coercion.

4. **`_request_has_webfetch = False` on `__init__`** — instance state
   on a CustomLogger that's typically a singleton across requests.
   This will misbehave under concurrent requests if any code path
   actually reads it. Worth confirming it's only ever consulted within
   the same hook callstack, or refactoring to thread through context.

5. **Naming.** `LITELLM_WEB_FETCH_TOOL_NAME = "litellm_web_fetch"` is
   fine, but the module is named `webfetch_interception` (no
   underscore) while the existing analogous module name pattern uses
   underscores consistently. Minor, but worth aligning.

I'd hold this until the streaming coercion is opt-in (or removed),
the docstring/default mismatch is resolved, and at least basic unit
tests exist. The feature itself is welcome — the surface area just
needs to be pinned down.
