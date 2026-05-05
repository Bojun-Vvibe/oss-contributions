# BerriAI/litellm #27219 — fix(mcp-semantic-filter): use chat schema when expanding tools for /v1/chat/completions

- Author: VANDRANKI
- Head SHA: `ff2cfa640ba7d5d2f80111fd62f9b3af5efd7a62`

## Files changed (top)

- `litellm/proxy/hooks/mcp_semantic_filter/hook.py` (+25 / -7)
- `litellm/responses/mcp/litellm_proxy_mcp_handler.py` (+4 / -1)

## Observations

- `litellm/responses/mcp/litellm_proxy_mcp_handler.py:391` — `target_format: Literal["responses", "chat"] = "responses"` correctly defaults to `"responses"` to preserve backward behavior for all existing callers of `_process_mcp_tools_to_openai_format`. Good.
- `litellm/proxy/hooks/mcp_semantic_filter/hook.py:67-78` — added `target_format: str = "chat"` default on `_expand_mcp_tools` is the load-bearing fix: this function is only invoked from the chat-completions hook path, so chat-as-default is the right choice. The new docstring `Args:` block is helpful.
- `:97` — `target_format=target_format,  # type: ignore[arg-type]` — the `# type: ignore` is an honest acknowledgement that `_expand_mcp_tools` accepts a wider `str` while the downstream `_process_mcp_tools_to_openai_format` expects a `Literal["responses", "chat"]`. Cleaner option: also type the local parameter as `Literal["responses", "chat"]` and drop the ignore; the casting cost is zero and the ignore disappears.
- `:181-186` — `_target_format = "responses" if call_type == "aresponses" else "chat"` — this is the runtime branch that disambiguates the two API shapes. Worth verifying `call_type` is actually `"aresponses"` for the responses path (vs e.g. `"responses"` or some other constant); a test pinning both arms would prevent future drift.
- No new test was added in this diff to cover the `call_type == "aresponses"` branch routing back to the `"responses"` format. The fix is a one-character change in semantics but the code path now has a conditional that wasn't there — at minimum a unit test asserting `_expand_mcp_tools` produces nested-`function` schema for chat call_type and flat schema for `aresponses` would lock the fix in.
- The hook docstring at `:69-78` correctly documents both formats and the default — nice.

## Verdict

`merge-after-nits`

Correct and surgical fix to a real schema-format mismatch (chat completions consumes nested `function` key, responses API consumes flat). Two nits before merge: (1) tighten the `_expand_mcp_tools` `target_format` parameter to `Literal["responses", "chat"]` so the `# type: ignore` at `:97` can be removed, and (2) add a regression test pinning the `call_type == "aresponses"` → `"responses"` format routing to prevent drift.
