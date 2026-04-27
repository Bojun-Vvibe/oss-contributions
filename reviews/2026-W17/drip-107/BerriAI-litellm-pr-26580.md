# BerriAI/litellm #26580 — fix(semantic-filter): only filter MCP tools, pass non-MCP through

- **Repo**: BerriAI/litellm
- **PR**: #26580
- **Author**: russellbrenner
- **Head SHA**: a23868d7f8643b5c9290ec5f9979609c555da2b3
- **Base**: main
- **Size**: +250 / −33 across two files:
  `litellm/proxy/hooks/mcp_semantic_filter/hook.py` (+67/-10) and
  `tests/test_litellm/proxy/_experimental/mcp_server/test_semantic_tool_filter.py`
  (+183/-23).

## What it changes

The `SemanticToolFilterHook` previously fed *every* tool in the
incoming request through the embedding-based filter
(`SemanticToolFilter.filter_tools`). That silently dropped
user-defined function tools and model-specific tool schemas (e.g.,
OpenAI `web_search_preview`, Anthropic `computer`) whenever the
embedding model decided they weren't a semantic match for the user
query.

This PR partitions tools by name prefix:

1. `hook.py:248-264` walks the request tools, splits them into
   `mcp_tools` (name starts with `mcp__` — the prefix the LiteLLM
   MCP gateway assigns to MCP-served tools) and
   `passthrough_with_positions` (everything else, paired with its
   original index).
2. If `mcp_tools` is empty, the hook short-circuits at
   `hook.py:266-271` (no embedding call, no header emission). This
   skips the cost of an embeddings round-trip for requests that
   only carry function tools.
3. `filter.filter_tools(...)` is called with `mcp_tools` only
   (`hook.py:283-287`).
4. Tool ordering is reconstructed by the new
   `_recombine_preserving_order` helper at `hook.py:131-159`. It
   iterates over the original positions and emits each pass-through
   tool at its original index; the filtered MCP block is anchored
   at the position of the first MCP tool in the input (subsequent
   MCP slots are absorbed into the filtered block).
5. The filter-stats header at `hook.py:295-296` now reports
   `len(mcp_tools)->len(filtered_tools)` instead of
   `original_tool_count->len(filtered_tools)`, so dashboards measuring
   "what fraction of tools did we drop" stop being polluted by
   pass-through counts.

The corresponding test fixture at
`test_semantic_tool_filter.py:366-378` is updated to use
`mcp__tool_<i>` names so the existing assertion
`len(filtered) < len(tools)` continues to exercise the filtering
path.

## Strengths

- The right defensive posture: the filter has no business making
  semantic-match decisions about tools the proxy didn't introduce.
  User-supplied function tools are part of the request contract;
  silently dropping them is a correctness bug, not a UX choice.
  The `mcp__` prefix is the proxy's own marker, so using it as the
  partition key is principled.
- The `_recombine_preserving_order` helper at `hook.py:131-159` is
  a static method with a clear contract spelled out in the
  docstring and is straightforward to unit-test in isolation
  (the test file gains 19-passing coverage according to the PR
  body). The "anchor at first MCP slot, absorb subsequent MCP
  slots" semantics is the only sensible choice — preserving the
  position of every original MCP slot would scatter the filtered
  block across the list and invent ordering that wasn't in the
  caller's request anyway.
- The early-return at `hook.py:266-271` when `mcp_tools` is empty
  removes a real cost: under "user has 12 function tools, no MCP
  tools" the previous code called the embeddings provider for
  every request. Now it returns immediately, no header, no
  embedding call.
- Removing `original_tool_count` (deletions at `hook.py:163` and
  `:189`) and re-deriving it as `len(mcp_tools)` keeps a single
  source of truth and lines up with what the header is meant to
  represent.
- The test fixture rename (`tool_<i>` → `mcp__tool_<i>`,
  `test_semantic_tool_filter.py:373-378`) is the minimum-impact
  change needed to keep the existing assertion valid; the
  surrounding mock-embedding logic (which decides what the
  embedding model "thinks" matches) is unchanged.

## Risks / nits

- The MCP-prefix detection at `hook.py:255-261` only checks
  `function.name` (for `dict` tools) or `name` (for object tools).
  The LiteLLM MCP gateway also assigns the prefix to `function`
  variant tools, but if a request comes in with a non-standard
  shape (e.g., `tool["type"] == "function"` but `function` is
  absent because the caller forgot to nest), the detection
  silently treats it as pass-through. Worth a defensive log line
  ("tool with no recognizable name field passed through
  unfiltered") so misshapen requests don't produce mysterious
  behavior.
- The "anchor at first MCP slot" rule is correct, but if a caller
  intentionally interleaves MCP and non-MCP tools (e.g., for a
  client that displays tools in groups in the order presented),
  the recombined output will collapse all MCP tools into a single
  contiguous block at the first MCP slot. That's a behavior
  change for a small but real class of caller. Worth calling out
  in the changelog.
- The expansion path at `hook.py:175-187` (where `litellm_proxy`
  references in `tools` are expanded into concrete MCP tool entries
  *before* the partitioning step) is preserved. After expansion the
  expanded entries should all carry the `mcp__` prefix, so they'll
  land in `mcp_tools`. Worth confirming with a test (the existing
  `test_semantic_tool_filter.py` doesn't appear to cover the
  expansion + recombination path together).
- The filter-stats header change (`mcp_count -> filtered_count`)
  is a metric-shape change. Dashboards that ingest this header and
  divide by it for "filter ratio" will see the denominator drop
  for any request that mixed MCP and non-MCP tools. Mention this
  in release notes.
- `_recombine_preserving_order` doesn't assert that
  `passthrough_with_positions` is sorted by `original_index`. It
  is, by construction (the loop at `:255-264` enumerates in
  order), but a defensive `assert sorted(...)` or `iter()`-with-
  `next` pattern (already used) is fine. A single-line
  precondition comment would future-proof against a refactor that
  reorders.
- The hook signature (`async_pre_call_hook`) is unchanged, so this
  is a behavior change inside an existing extension point. If
  downstream forks have monkey-patched the previous "filter
  everything" behavior, they'll need to re-validate.

## Suggestions

- Add a test case combining the `_should_expand_mcp_tools` path
  (tools containing `server_url == "litellm_proxy"`) with the new
  partitioning, so expansion + filter + recombine is exercised
  end-to-end.
- Add a test for the mixed-tool case where MCP and non-MCP tools
  are interleaved in the request, to pin the "anchor at first MCP
  slot, absorb subsequent slots" ordering rule.
- Consider exposing the prefix `"mcp__"` as a module-level
  constant (or importing it from wherever the gateway *assigns*
  the prefix) so the partitioning predicate stays in sync if the
  prefix ever changes.
- Emit a debug-level log when a request carries no MCP tools and
  the hook short-circuits, so operators tracing "why didn't this
  request hit the embedder" have a one-line trail.

## Verdict

`merge-after-nits` — the partition fixes a real silent-drop
correctness bug, the recombination preserves caller ordering for
the common case, and the new short-circuit is a free latency
win. Add the expansion-path integration test and surface the
`"mcp__"` constant before merge; the metric-shape change in the
filter-stats header should be flagged in release notes.
