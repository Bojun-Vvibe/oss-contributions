# BerriAI/litellm#26887 — Fix Vertex web search with function tool conflict

- PR: https://github.com/BerriAI/litellm/pull/26887
- Head SHA: `e33fca32b4d53c524467d8dd92922ad689897555`
- Author: Genmin
- Files: 2 changed (`vertex_and_google_ai_studio_gemini.py` +56,
  `tests/.../test_vertex_and_google_ai_studio_gemini.py` +139/−8)
- Fixes #26882

## Context

Vertex Gemini rejects requests that mix `function_declarations` tools with
search tools (`googleSearch`, `google_search_retrieval`,
`enterpriseWebSearch`, `urlContext`) with the error
`Multiple tools are supported only when they are all search tools`. The
config's `_map_function` already drops search tools when it sees them
alongside function declarations, but `web_search_options` is mapped via a
*separate* path (`_map_web_search_options`) and gets appended to
`optional_params["tools"]` later — so a request with `tools=[function_tool]`
plus `web_search_options={...}` would produce two Tool objects on the wire
and trigger the Vertex error. Order-of-processing-dependent: the bug
manifests differently depending on whether `tools` or `web_search_options`
is processed first.

## Design

The fix moves the conflict resolution out of `_map_function` (which only
runs on the function path) and into the *append point* —
`_add_tools_to_optional_params` — so it runs regardless of which path put
tools into `optional_params` first.

**New helpers** (`vertex_and_google_ai_studio_gemini.py:539-572`):

- `_has_function_declarations_tool(tools)` returns True if any tool dict
  contains a `function_declarations` or `functionDeclarations` key. The
  camelCase fallback handles requests that already use the Vertex wire
  shape directly.
- `_is_search_tool(tool)` checks the tool dict against
  `search_tool_keys = {GOOGLE_SEARCH, GOOGLE_SEARCH_RETRIEVAL,
  ENTERPRISE_WEB_SEARCH, URL_CONTEXT, "google_search",
  "google_search_retrieval", "enterprise_web_search", "urlContext"}` —
  both the `VertexToolName` enum values *and* the bare snake_case strings
  (which is what `_map_web_search_options` emits).
- `_drop_search_tools_if_mixed_with_functions(optional_params)`: the
  guard. It bails if there are no function declarations (`:567-569`),
  bails if `include_server_side_tool_invocations` is set (`:572-575`,
  the Gemini 3+ escape hatch), filters out search tools, and warns if
  any were dropped.

**Append-point hook** (`:589-595`):
```python
def _add_tools_to_optional_params(self, optional_params: dict, tools: List) -> dict:
    optional_params = super()._add_tools_to_optional_params(
        optional_params=optional_params, tools=tools
    )
    self._drop_search_tools_if_mixed_with_functions(optional_params=optional_params)
    return optional_params
```

This runs *after* the parent class's tool-merge, which is the right
ordering: the parent unifies `optional_params["tools"]` with the new
`tools` list, then the conflict guard observes the post-merge state and
prunes. Two append calls, two prune passes — one of them is a no-op every
time, but both are required because either ordering can be the one that
triggers the conflict.

## Tests are the second-most-interesting part

Three new tests, one per ordering branch:

- `test_vertex_ai_web_search_options_after_function_tools_drops_search`
  (`:3917-3955`): function tools first, then `web_search_options`. After
  both `_add_tools_to_optional_params` calls, asserts
  `len(optional_params["tools"]) == 1` and the surviving tool is the
  function declaration.
- `test_vertex_ai_web_search_options_before_function_tools_drops_search`
  (`:3957-3994`): the reverse ordering. Same final assertion. This is the
  test that proves the fix is order-independent — the search tool gets
  added first, and the *second* append (function tools) is what triggers
  the prune.
- `test_vertex_ai_web_search_options_with_function_tools_preserved_for_server_side_invocations`
  (`:3997-4035`): with `include_server_side_tool_invocations=True`,
  asserts both `function_declarations` and `googleSearch` survive. This
  pins the Gemini 3+ escape hatch.

## Risks / nits

- **`_is_search_tool` key set is duplicated** between the enum values and
  the bare strings. If a new search tool is added (e.g.,
  `google_search_v2`), it has to be added in two places — the enum *and*
  this set. A `{*VertexToolName.search_tools(), *legacy_aliases()}`
  factory or a `_search_tool_key` set that derives from the enum would
  avoid that drift. Not a blocker; the current set is small.
- **Warning is `verbose_logger.warning(...)`** with a static message
  (`:580-585`). A user with a long-running proxy will see this on every
  affected request — fine for debugging, potentially noisy for production
  workloads where the user knows their tools mix and just wants the
  search dropped silently. A `warnings.warn(..., stacklevel=2)` once-per-
  process pattern would be tidier, but `verbose_logger` is the project
  convention so this is consistent.
- **The `super()._add_tools_to_optional_params` contract** — the base
  class's signature isn't shown in this diff window. Worth confirming
  that overriding this method doesn't break any other Vertex subclass
  that already overrides it; spot-checking the file (the diff context
  shows `_resolve_search_tool_conflict` already lives on this class) it
  looks like `VertexGeminiConfig` is the right level.

## Verdict

**`merge-as-is`**

The fix is at the architecturally correct point (tool-append, not
function-mapping), order-independent (proved by the two-direction tests),
and preserves the documented Gemini 3+ escape hatch. Test-to-fix ratio is
139:56 with three orthogonal scenarios pinned. The duplicated key-set is
minor enough to address in a follow-up.

## What I learned

When a transformation has multiple input paths converging on a shared
output (`optional_params["tools"]`), the right place to enforce a
cross-path invariant is at the convergence point, *not* at any individual
input path. The pre-PR code put the conflict check inside `_map_function`,
which assumed the function path was the last writer — an assumption that
was true until `_map_web_search_options` was added later. The
post-PR shape (guard at `_add_tools_to_optional_params`) is robust against
any future input path that lands a tool object — `_map_code_execution`,
`_map_grounding`, whatever comes next. The two-direction test pattern is
the cheapest possible proof of order-independence.
