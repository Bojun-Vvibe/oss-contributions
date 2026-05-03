# BerriAI/litellm PR #27088 — feat: WebFetch interception with Firecrawl support

- Link: https://github.com/BerriAI/litellm/pull/27088
- Head SHA: `60216bad222bfb4f08e72a02ef81d45f83ad0bf5`
- Author: menardorama
- Size: +2492 / −2 (18 files, 11 new)

## Files changed (highlights)
- `litellm/integrations/webfetch_interception/handler.py` — new (1043 lines, the agentic loop)
- `litellm/integrations/webfetch_interception/transformation.py` — new (343 lines)
- `litellm/integrations/webfetch_interception/tools.py` — new (181 lines)
- `litellm/llms/firecrawl/fetch/transformation.py` — new (128 lines, FirecrawlFetchConfig)
- `litellm/llms/base_llm/fetch/transformation.py` — new (85 lines, BaseFetchConfig)
- `litellm/proxy/proxy_server.py` — `+70 / −2` (parse_fetch_tools)
- `litellm/router.py` — `+4 / −0` (`fetch_tools` parameter)
- `tests/webfetch_tests/test_webfetch_interception.py` — new (390 lines, 19 tests)
- `tests/webfetch_tests/test_firecrawl_fetch.py` — new (123 lines, 8 tests)

## Reasoning

This is a substantial new subsystem. The architecture explicitly mirrors the existing `websearch_interception/` module — that's the right call (consistency, cross-tool maintainability) and it's the strongest argument in this PR's favor. The author has clearly studied the existing pattern.

Concrete observations:

1. **Default-on for Bedrock is surprising.** `litellm/integrations/webfetch_interception/handler.py:71` shows:
   ```
   if enabled_providers is None:
       self.enabled_providers = [LlmProviders.BEDROCK.value]
   else:
       self.enabled_providers = [...]
   ```
   The docstring above it says "If None or empty list, enables for ALL providers. Default: None (all providers enabled)" — but the code defaults to Bedrock-only. Either the docstring or the code is wrong; given that turning interception on globally would be surprising, the *code* is probably right and the *docstring* needs to match.

2. **Streaming silently disabled.** At `litellm/integrations/webfetch_interception/handler.py:144`:
   ```
   if kwargs.get("stream"):
       verbose_logger.debug(
           "WebFetchInterception: deployment hook converting stream=True to stream=False"
       )
       kwargs["stream"] = False
       kwargs["_webfetch_interception_converted_stream"] = True
   ```
   This converts a streaming request into a non-streaming one without surfacing it to the caller. A client expecting an SSE stream will hang waiting for chunks that never come, then get one fat response. This needs to either:
   - Re-stream the final response back to the caller (preferred), or
   - Document loudly that enabling this callback breaks streaming for any conversation that includes a `web_fetch` tool.

3. **1043-line `handler.py`.** This is too much for a single file. The agentic loop, the deployment hook, the pre-request hook, and the config-loading classmethod should each live in their own module — that's how the comparable `websearch_interception/` module is laid out (per the PR description). Without that decomposition the file is effectively unreviewable in detail.

4. **Tool-name conflict risk.** `litellm/constants.py:516` adds `LITELLM_WEB_FETCH_TOOL_NAME = "litellm_web_fetch"`. If a downstream user has already defined a tool called `litellm_web_fetch` for some other purpose, every request now collides. Worth at least documenting the reserved name in the proxy config docs.

5. **27 tests is good coverage on paper, but the integration tests live in `tests/webfetch_tests/`** (a new top-level test dir not seen in the existing suite). Need to confirm CI is configured to discover that directory — `tests/test_litellm/` and `tests/local_testing/` are the conventional homes.

6. **No test for the `is_web_fetch_tool` substring/type detection in `tools.py`** that I can see in the diff snippet. Given that misidentification here would either silently bypass interception or hijack unrelated tools, this is the highest-value test surface.

7. **`router.py: +4 / −0`** to add `fetch_tools` is correctly minimal, but mirrors a pattern that already requires backwards-compat handling for older proxy configs that don't define `fetch_tools`. Confirm that an existing config without `fetch_tools` and without the new callback continues to work unchanged.

## Verdict

`needs-discussion`

The feature is well-motivated and the architecture is the right one (it mirrors the established websearch interceptor). But this PR can't land as-is — too many unresolved questions:
- Streaming-silent-downgrade behavior is a real correctness regression for any caller that uses SSE.
- Default provider scope contradicts its own docstring.
- 1043-line handler module needs to be decomposed before reviewers can give it a meaningful line-by-line.
- CI test discovery for `tests/webfetch_tests/` needs explicit confirmation.

Recommend the author respond to these before a real diff-level review pass.
