# BerriAI/litellm #26641 — fix(mcp): preserve native tools in semantic filter hook

- PR: https://github.com/BerriAI/litellm/pull/26641
- Head SHA: `2e1a61dced7013cabbaf64f5ba146839bda21d4b`
- State: CLOSED
- Author: ayushh0110 (Ayush Shekhar)
- Files: `litellm/proxy/hooks/mcp_semantic_filter/hook.py` (+115/−25), `tests/test_litellm/proxy/_experimental/mcp_server/test_semantic_tool_filter.py` (+238/−21)
- Fixes: #26212

## Observations

- Real bug: `SemanticToolFilterHook.async_pre_call_hook` was passing **all** tools (MCP-registered + user-supplied native tools) to `filter_tools()`, which queries a `SemanticRouter` whose `_tool_map` is built **exclusively** from the MCP registry. Native tools have no entry in `_tool_map`, so `_get_tools_by_names()` silently drops them. The LLM never sees native tools → either responds in prose or the upstream provider 400s on a tool-only request that arrives toolless. This is the canonical "filter widened beyond its source-of-truth" bug class.
- Fix shape (correct): partition the input tools into native vs MCP-registered **before** filtering. Run semantic filter only on the MCP set; merge native tools back unconditionally on the way out. Net effect: native tools are guaranteed to survive, MCP filtering happens on its own legitimate input set.
- The `_is_mcp_tool()` helper uses **shape-based detection** for OpenAI-format dicts: `isinstance(tool, dict) and "function" in tool and "type" in tool`. PR explicitly says "safe regardless of future `_extract_tool_info` changes" — this is the right discipline (don't depend on the brittle path through `_extract_tool_info`'s name-extraction).
- `tools_expanded_from_mcp` flag tracks whether MCP tools have been expanded into per-function entries. **Load-bearing detail**: without this flag, expanded MCP tools (which post-expansion look indistinguishable from native tools at the dict level) would be misclassified as native and silently bypass semantic filtering. PR correctly anticipates this trap.
- `filter_stats` reporting flips from "total → kept" (e.g. `9->6`) to **MCP-only** counts (e.g. `5->2`). This is the right semantic for an observability surface — the metric should report what the filter actually filtered, not include rows that flowed around it.
- `_emit_filter_metadata()` extracted as a helper. Skipping spurious `semantic_filter_*` response headers when the request is all-native is a small but correct polish — observability shouldn't lie about a filter being active when it never ran.
- Test surface (18 tests, 9 existing + 7 `TestGetToolsByNames` + 2 new):
  - `test_semantic_filter_hook_preserves_native_tools` — mixed MCP + native: native survives, `filter_stats` reports MCP-only counts. **The discriminating cell** that pins the bug fix.
  - `test_semantic_filter_hook_all_native_tools` — all-native: all pass through, no spurious filter headers emitted. Pins the metadata polish.

## Risks / nits

- The shape-based `_is_mcp_tool()` is right *for OpenAI-format dicts*. What about Anthropic-style tool entries (no `type` field, just `name`/`description`/`input_schema`)? If any LiteLLM provider transforms native tools into a non-OpenAI dict shape *before* this hook runs, those would be misclassified as MCP (no `type` field → not native) and then dropped from the filter pass-through. Worth a discriminating cell for Anthropic-format input.
- `tools_expanded_from_mcp` flag — where does it get set? Need to verify it's set on **every** code path that expands MCP tools, not just the common one. A single missed expansion site re-introduces the bypass.
- `filter_stats` change from total-counts to MCP-only-counts is a **breaking change** for downstream observability consumers who were reading the old `9->6` semantic. Probably nobody, but worth a CHANGELOG line.
- 7 new `TestGetToolsByNames` tests — the PR description doesn't enumerate them, so it's hard to know what they pin. Worth listing them in the PR body.
- The PR is **CLOSED**, not merged. Reasons: typically (a) superseded by another PR, (b) maintainer rejected the approach, (c) author abandoned. Reviewer should check the conversation thread to understand why before considering this for cherry-pick or re-submission.

## Verdict: `request-changes`

**Rationale:** The fix shape is correct and the bug is real, but the PR is in a CLOSED state — meaning maintainers and/or author concluded this isn't the merge path. Before this could ship, three things need clarification: (a) **why was it closed** (superseded? rejected approach? need to read the thread), (b) the **Anthropic-format tool shape** isn't covered by `_is_mcp_tool()` shape detection — needs a discriminating test cell, and (c) the `tools_expanded_from_mcp` flag needs a code-side audit confirming it's set on every MCP-expansion site, not just the common one. The bug class is well-identified ("filter widened beyond its source-of-truth") but the implementation has at least one un-tested input shape and an unexplained close.
