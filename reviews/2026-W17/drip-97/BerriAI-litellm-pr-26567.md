# BerriAI/litellm#26567: fix(semantic-filter): contribute upstreamed fixes

- **Author:** russellbrenner (russ)
- **HEAD SHA:** 8576b8be
- **Verdict:** merge-after-nits

## Summary

A +32/-8 single-file PR to
`litellm/proxy/hooks/mcp_semantic_filter/hook.py` that bundles two
related correctness fixes into the `SemanticToolFilterHook`. Fix 1:
adds `"anthropic_messages"` to the `call_type` allowlist at
`hook.py:154` so requests through `/v1/messages` (used by Claude
Code and other Anthropic-SDK clients) actually run the semantic
filter instead of being skipped. Fix 2: rewrites the tool-filtering
loop at `hook.py:221-260` to partition the incoming `tools` list
into `mcp_tools` (name-prefixed `mcp__`, assigned by the LiteLLM
proxy MCP gateway) and `passthrough_tools` (everything else —
user-defined function tools, model-specific schemas), filter only
the MCP slice, and recombine before write-back. Both fixes are
genuine bug fixes with clear failure modes (silent skip → no
filtering ever applied; or non-MCP tools getting dropped on the
floor by an indiscriminate semantic filter), and the diff
preserves the existing metadata-header emission semantics with
`filter_stats` correctly recomputed from `len(mcp_tools)` rather
than `original_tool_count`.

## Specific feedback

- `hook.py:154` — `call_type` allowlist now includes
  `"anthropic_messages"`. Verify against the call-type enum/string
  table that this is the correct identifier upstream produces for
  `/v1/messages` requests; a typo here silently re-disables the
  hook for that endpoint with no error surface. A grep for
  `"anthropic_messages"` across the rest of the proxy hook surface
  would confirm the spelling is canonical.
- `hook.py:223-237` — the partition loop:
  ```python
  name = (
      t.get("function", {}).get("name", "") or t.get("name", "")
      if isinstance(t, dict)
      else getattr(t, "name", "")
  )
  ```
  Two correctness concerns: (1) operator precedence — the
  ternary `if isinstance(t, dict) else ...` binds tighter than
  the `or`, so the dict branch evaluates
  `t.get("function", {}).get("name", "") or t.get("name", "")`,
  which is what's wanted, but it's worth a parenthetical for
  clarity: `(t.get("function", {}).get("name", "") or
  t.get("name", "")) if isinstance(t, dict) else ...`.
  (2) `getattr(t, "name", "")` returns a string for the
  non-dict branch but `t.name` could plausibly be a non-string
  on a Pydantic-model tool — coerce with `str(getattr(t, "name",
  "") or "")` to avoid an `.startswith` AttributeError in that
  edge case.
- `hook.py:240-244` — the early-return when `mcp_tools` is empty
  is correct (no filtering needed when nothing is filterable),
  but it returns `None` which by the surrounding contract means
  "no changes". Confirm that's the right signal vs. returning
  `data` unchanged; both can be valid depending on caller
  contract but they're not interchangeable in some hook frameworks.
- `hook.py:259` — `data["tools"] = filtered_tools + passthrough_tools`
  changes ordering: filtered MCP tools come first, passthrough tools
  second. If any caller depends on the original tool order
  (e.g., for tool selection priority hints) this is a behavior
  change. Worth a comment that reordering is intentional and
  acceptable, or preserve original positions via index tracking.
- `hook.py:262` — `filter_stats = f"{len(mcp_tools)}->{len(filtered_tools)}"`
  changes the semantic of the stats header from "total->filtered"
  to "mcp_total->mcp_filtered". Worth a one-line note in the
  emitted header (or in the docs) clarifying the new
  interpretation, otherwise dashboards that consumed the old
  header value will silently shift.
- **Missing test coverage:** the PR pre-submission checklist
  explicitly requires "Adding at least 1 test is a hard
  requirement" but the file list shows only the hook source
  file, no test file under `tests/test_litellm/`. Block on a
  test for both fixes: (a) `anthropic_messages` call_type now
  hits the filter; (b) non-MCP tools survive when MCP tools
  are filtered.

## Risks / questions

- AI-attributed changes: the PR body is explicit that this is
  AI-generated co-authored work. The fixes are small and the
  logic is verifiable by inspection — not a blocker, but the
  test requirement above is doubly important here.
- Multi-tool tool-call schema variants: this code path runs
  before the LLM call, so it sees both OpenAI-style (`{"type":
  "function", "function": {"name": ...}}`) and Anthropic-style
  (`{"name": ..., "input_schema": ...}`) tool dicts. The
  partition loop handles both via the dual-key lookup, which is
  good; confirm it also handles Bedrock-Converse-style
  (`{"toolSpec": {"name": ...}}`) if that's in scope for this
  proxy.
- `tool_names_csv` derivation: confirm the helper handles a mix
  of dict-typed and object-typed entries correctly post-merge of
  filtered + passthrough lists.
