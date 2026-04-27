# BerriAI/litellm #26634 — fix(mcp): preserve native tools in semantic filter hook

- URL: https://github.com/BerriAI/litellm/pull/26634
- Head SHA: `1b0ef5f9d7756c658c8ae4632be72b1ea6a12a3f`
- Diff: +290/-41 across `litellm/proxy/hooks/mcp_semantic_filter/hook.py` and `tests/test_litellm/proxy/_experimental/mcp_server/test_semantic_tool_filter.py`. Closes #26212.

## Context / problem

`SemanticToolFilterHook.async_pre_call_hook` was passing **all** tools (MCP-registered + native OpenAI-format function tools) to `self.filter.filter_tools()`, which queries a `SemanticRouter` whose `_tool_map` is built exclusively from the MCP registry at startup. Native tools have no entry in `_tool_map`, so the router's `_get_tools_by_names()` silently dropped them. Net effect: a request that mixes native function tools + MCP tools loses all native tools after the hook runs, the LLM either responds in prose ("I would call your function...") or the upstream provider returns 400, and there's no diagnostic in the response — the tool count stat reads `7→5` and looks like normal filtering.

## What the fix does

`hook.py:126-141` adds a `_is_mcp_tool(tool: Any) -> bool` helper that calls `self.filter._extract_tool_info(tool)` and checks `name in self.filter._tool_map`. The docstring correctly documents the OpenAI-format-dict contract: `_extract_tool_info` returns `""` for `{"type":"function","function":{"name":...}}` shapes because the name is nested under `function`, not top-level — so empty-string-not-in-`_tool_map` is the load-bearing piece that classifies native function tools as native.

`hook.py:236-289` partitions: `native_tools = [t for t in tools if not self._is_mcp_tool(t)]`, `mcp_tools = [t for t in tools if self._is_mcp_tool(t)]`. Filter runs only on `mcp_tools` (skipped entirely if empty); result is `native_tools + filtered_mcp_tools`. Response-header metadata (`litellm_semantic_filter_stats`, `litellm_semantic_filter_tools`) is gated behind `if mcp_tools:` so all-native requests don't emit spurious `N->N` stats. Logging is enhanced to report MCP/native counts separately at info+debug levels.

## Specific references

- `hook.py:126-141` — `_is_mcp_tool` helper. Critical docstring NOTE block names the load-bearing invariant: `_extract_tool_info` returning `""` for OpenAI-format dicts is *what makes the classification work*. If `_extract_tool_info` is ever changed to unwrap nested `function.name`, native tools would suddenly classify as MCP (with an empty-string `_tool_map` lookup that... still misses, so still classifies as native — but the docstring's "must be revisited" warning is right because the behavior could become subtly version-dependent on the SemanticRouter implementation).
- `hook.py:241-243` — partition. Two list comprehensions over `tools` is O(2n) — acceptable for typical tool counts (10-100) but if `_is_mcp_tool` ever does anything expensive, fold to a single pass.
- `hook.py:255-261` — empty-MCP-list short-circuit. Correct: `self.filter.filter_tools([])` would presumably no-op anyway, but skipping avoids the embedding model call entirely.
- `hook.py:269-281` — `if mcp_tools:` gate on the response-header metadata. Closes the spurious-stats hole the PR body calls out as a P2.
- Tests at `test_semantic_tool_filter.py:+220` cover the two new cases (`test_semantic_filter_hook_preserves_native_tools` mixed 5+2, `test_semantic_filter_hook_all_native_tools` all-native pass-through). PR body claims 18 tests pass total (9 existing + 2 new + 7 TestExtractToolInfo); the 7 extras presumably pin `_extract_tool_info`'s return-shape contract that `_is_mcp_tool` depends on, which is exactly the right defensive coverage given the docstring NOTE.

## Risks / nits

- Pre-Submission checklist: `make test-unit` checkbox shape in PR body uses `[ ] - [x]` (nested-bracket typo) on three rows including the `make test-unit` row. Maintainers should confirm CI actually ran.
- The `_extract_tool_info` contract dependency is a real fragility — would be cleaner if `_is_mcp_tool` did its own structural check on `tool.get("type") == "function"` and `tool["function"]["name"] in _tool_map_with_function_names`, rather than depending on the `""` return value of an internal helper. But that's a refactor, not a blocker.
- Two-list-comp partition is O(2n); a single-pass `for t in tools: (mcp if self._is_mcp_tool(t) else native).append(t)` is marginally better. Negligible at typical tool counts.

## Verdict

`merge-as-is` — root-cause fix with the right partition shape, the right gate on response-header emission, paired regression coverage for both mixed and all-native cases, and a docstring that names the load-bearing assumption. The Pre-Submission checkbox typo is cosmetic.
