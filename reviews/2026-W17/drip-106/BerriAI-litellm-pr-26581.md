# BerriAI/litellm PR #26581 — fix(semantic-filter): run for anthropic_messages call type

- **Link**: https://github.com/BerriAI/litellm/pull/26581
- **Head SHA**: `0b86ca211ccf76921f888a80a041ccb509925be5`
- **Author**: russellbrenner
- **Size**: +118 / -23 across 2 files
- **Verdict**: `merge-as-is`

## Files changed
- `litellm/proxy/hooks/mcp_semantic_filter/hook.py:150-167` — extends the call-type allowlist in `async_pre_call_hook` from `("completion", "acompletion", "aresponses")` to also include `"anthropic_messages"`. Includes a 4-line load-bearing comment explaining that this is the call type used by `/v1/messages` requests (Claude SDK clients) and that without the entry, MCP tool filtering silently no-ops for that whole client class.
- `tests/test_litellm/proxy/_experimental/mcp_server/test_semantic_tool_filter.py` — net +103 lines, mostly a new `_AnthropicMessagesMockFilter` stub plus parametrized tests in a new `# SemanticToolFilterHook — anthropic_messages call type allowlist` section. Existing test bodies are reformatted (whitespace-only) to satisfy the project's formatter.

## Analysis

This is the smallest possible correct fix for a real correctness bug. The core change is a 5-line allowlist addition (`"anthropic_messages"` added to the tuple) plus a comment that explains *why* the omission was a silent functional regression rather than a missing feature. That comment is the right kind of PR-prose-in-code: anyone running `git blame` on the tuple in 6 months will immediately understand why the entry exists, which is the only defense against a future "consolidation" PR yanking it back out as "presumably dead code."

The bug shape is straightforward but worth naming: the semantic filter is supposed to narrow the MCP tool list to a top-K that matches the user query before forwarding tools to the model, both to reduce token cost and to reduce model confusion from irrelevant tools. The pre-fix `if call_type not in (...)` guard short-circuited to `verbose_proxy_logger.debug(...)` and returned None for any call type not in the allowlist — meaning every `/v1/messages` request from a Claude Code or Anthropic SDK client received the *full* unfiltered MCP tool catalog. That's not just a missing-feature; it's a silently-broken contract because the filter was *configured* and the user reasonably believed it was running.

The test addition is shaped well:
- A purpose-built `_AnthropicMessagesMockFilter` stub rather than reusing the OpenAI-style mock — necessary because Anthropic-style messages carry typed content blocks (`{type: "text", text: "..."}`) rather than flat strings, and `extract_user_query` has to walk that shape via `for m in reversed(messages)`.
- The reverse iteration in `extract_user_query` is correct: the most recent user message is the query relevant for top-K matching.
- The reformatting noise in the existing tests (collapsing `_get_tools_by_names(\n    ["foo"], available_tools\n)` to `_get_tools_by_names(["foo"], available_tools)`) is unfortunate and inflates the diff but is the project's prevailing style — not a blocker.

## Why merge-as-is

1. The fix is minimal and additive (one tuple entry).
2. The comment names the failure mode in operator-meaningful terms ("every MCP tool is forwarded on every request").
3. Tests are added in the correct location and exercise the new path with the right message shape.
4. No call-site upstream of `async_pre_call_hook` needs to change — the new call type was already passed through, it just wasn't being honored.
5. The author's "PR scope is as isolated as possible" checkbox is genuinely true: the only non-allowlist change is whitespace from the formatter.

## Optional follow-ups (not blocking)

- The `verbose_proxy_logger.debug(f"Skipping semantic filter for call_type={call_type}")` log line at the rejection branch is fine but, for the next time someone adds a new call type, would benefit from a one-line list of *valid* call types in the message body so the operator can immediately see whether their call type is supposed to match. Consider `f"Skipping semantic filter for call_type={call_type}; supported types: {SUPPORTED_TYPES}"` once the tuple grows large enough to warrant being a module constant.
- Sibling MCP hooks (`mcp_pre_call_hook`, etc.) likely have the same allowlist drift — worth a one-line sweep with `rg 'call_type not in' litellm/proxy/hooks/` as a follow-up audit.
- A `text_completion`-style call type test for the *negative* case (filter correctly skips for legacy Anthropic completions API) would lock the boundary tighter, but the existing skip-for-unknown-type test probably already covers it.

## What I learned

The "configured-but-silently-inactive" failure mode is the worst kind of misconfiguration because the operator gets no signal: the hook is registered, the config validates, no errors fire. The fix here doesn't add a louder warning for unknown call types (which would be a larger debate — fail-loud vs fail-open for hook registration), but it does add the comment that prevents the regression from recurring. For a project with multiple parallel call-type surfaces (`/chat/completions`, `/responses`, `/messages`), maintaining an authoritative-types registry that hooks subscribe to by name might be a worthwhile architectural follow-up; the current "tuple of strings in each hook" pattern guarantees this drift will recur.
