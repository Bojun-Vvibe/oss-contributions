# BerriAI/litellm#26858 — fix(mcp_semantic_tool_filter): resolve client-suffixed tool names (LibreChat-style)

- PR: https://github.com/BerriAI/litellm/pull/26858
- Head SHA: `4dcb8c5ccc377f85e2f31ea3fcb6b6f9d07e1ec8`
- Author: GopalGB
- Closes: #26507
- Files: 100 changed, +8649 / −630 (the *focused* change is ~80 lines in
  `semantic_tool_filter.py` + new tests; the other 99 files are branch-drift from
  rebasing onto a fast-moving main with parallel migrations and unrelated
  refactors. **This PR badly needs a rebase to a clean diff before merge.**)

## Context

The semantic-tool filter sits between an MCP client (e.g. an LLM-driven UI that
talks the MCP spec) and the proxy's MCP server. Different clients wrap the
proxy's canonical tool names differently to disambiguate same-named tools
across multiple connected servers:
- **opencode-style:** prefix wrapping — `<client_alias><sep><canonical>`,
  e.g. `litellm_fc_web_search-firecrawl_scrape`
- **LibreChat-style:** suffix wrapping — `<canonical><sep><client_uid>`,
  e.g. `fc_web_search-firecrawl_scrape_a1b2c3d4`

Pre-PR, `_name_matches_canonical` only handled the prefix shape. When a
LibreChat-style client connected, suffix-wrapped tool names fell through the
matcher, the filter shipped `tools: []` with `tool_choice: "auto"` to the
upstream provider, and every strict OpenAI-compatible model returned
`400 'tool_choice' is only allowed when 'tools' are specified`. So the
semantic filter was actively breaking LibreChat-shaped traffic.

## Design

**Matcher (`semantic_tool_filter.py:213-263`).** `_name_matches_canonical` now
checks both directions, anchored to a separator on the boundary side
(underscore or dash) regardless of `MCP_TOOL_PREFIX_SEPARATOR`:

```python
if client_name == canonical:
    return True
# Suffix match: client wraps as ``<alias><sep><canonical>``.
if client_name.endswith(canonical):
    separator = client_name[-len(canonical) - 1]
    if separator in ("_", "-"):
        return True
# Prefix match: client wraps as ``<canonical><sep><uid>``.
if client_name.startswith(canonical):
    separator = client_name[len(canonical)]
    if separator in ("_", "-"):
        return True
return False
```

Both branches are still gated on the early check that `canonical` itself
contains `MCP_TOOL_PREFIX_SEPARATOR` (lines around 251) — the docstring
correctly preserves the rationale: server-registered MCP tools are always
emitted as `<server_name><MCP_TOOL_PREFIX_SEPARATOR><tool_name>` (see
`add_server_prefix_to_name`), so a canonical without the separator is not a
namespaced MCP tool, and falling back to either prefix or suffix matching
would spuriously collide with unrelated user-defined local functions whose
names happen to share the same characters.

This is the right gate. Without it, a client function literally named
`scrape` would match canonical `firecrawl-scrape` via the prefix-wrap
branch and get silently substituted.

**Selector (`semantic_tool_filter.py:280-330`).** `_get_tools_by_names` now
documents and implements that "exact matches win over wrapped matches" and
"shortest qualifying name wins among multiple wrap-compatible candidates":

```python
# Prefer the shortest qualifying name. When several
# incoming tools wrap the same canonical (prefix or
# suffix wrapping), the one closest in length to the
# canonical is the least-wrapped and most likely the
# intended target.
```

The "shortest = least-wrapped" tiebreaker is sound for both wrap styles
because the wrapping is purely additive on either end. The docstring
update (lines 269-279) is the right comment to land alongside.

**Server-name resolution (`server.py:715-720`).** A separate small change
factors out `iter_known_server_prefixes(server)` (now imported from line 52)
to enumerate `[server.alias, server.server_name, server.server_id]` for the
allowed-server lookup. Pure refactor, no behavior change at the
match-list level. Worth landing.

**Tests.** New cases in `tests/test_litellm/proxy/_experimental/mcp_server/
test_semantic_tool_filter.py` cover the suffix-wrap shape (the LibreChat
repro `fc_web_search-firecrawl_scrape_a1b2c3d4` matching canonical
`fc_web_search-firecrawl_scrape`). Most of the rest of the test diff is
formatter-driven line collapsing (long `assert ..., f"..."` lines unfolded
into single lines), which adds noise without changing behavior.

## Risk analysis

**Massive collateral diff.** This is the biggest concern by far. Of 100
files touched, roughly 4 are the actual fix:
- `litellm/proxy/_experimental/mcp_server/semantic_tool_filter.py`
- `litellm/proxy/_experimental/mcp_server/server.py`
- `tests/test_litellm/proxy/_experimental/mcp_server/test_semantic_tool_filter.py`
- (test fixtures, if any)

The other ~96 are unrelated (`migrations/20260429120000_search_tools_on_object_permission`,
`migrations/20260429161855_workflow_runs_tables`, `.npmrc`, GH workflow
files, `schema.prisma`, `constants.py`, embedding/streaming/cost-calc
internals…). This is almost certainly accidental from a rebase or from
branching off a stale main — but **shipping it as one PR makes the diff
unreviewable line-by-line** and risks dragging in unrelated issues
(e.g. unrelated migrations applying out-of-order on customer environments
that pull this PR into a branch).

**Symmetric ambiguity.** Consider canonical tools `foo` and `foo_bar` both
exposed by the proxy with separator `_`. A client suffix-wrap presents
`foo_bar_xyz`. The matcher will match canonical `foo_bar` (prefix branch:
starts with `foo_bar`, separator at position 7 is `_`) — correct. But it
also returns True for canonical `foo` (prefix branch: starts with `foo`,
separator at position 3 is `_`) — *also* correct under the matcher's
contract, which is purely "is wrap-compatible." The selector then
disambiguates via the shortest-qualifying-name tiebreaker. The selector's
tiebreaker semantics need at least one test case asserting the right tool
wins in this collision shape. The current test set repros the *single-tool*
LibreChat case — good — but doesn't pin the *multi-tool* tiebreaker.

**No bound on `_get_tools_by_names` cost.** With both wrap directions
considered, the inner loop scans `available_by_name` per canonical. For a
client connecting hundreds of tools across a dozen servers this stays
small, but worth noting that the prior matcher had one comparison per
candidate; the new matcher does two `endswith`/`startswith` plus separator
checks. Negligible in practice.

## Verdict

**`request-changes`**

The fix itself is correct, well-documented, and gated correctly against the
spurious-collision class via the canonical-contains-separator early check.
I'd happily approve the focused change. But this PR cannot be merged in
its current shape:

1. **Rebase to isolate the actual fix.** Drop the 96 unrelated files
   (`migrations/*`, `.npmrc`, workflow YAMLs, unrelated `schema.prisma`
   columns, `constants.py`, etc.). The reviewer-readable diff should be
   `semantic_tool_filter.py` + `server.py` (the small `iter_known_server_prefixes`
   factoring) + the new tests. Everything else either belongs in another PR
   or has already merged on main and is showing as a "diff" only because
   this branch hasn't been refreshed.
2. **Add a multi-tool ambiguity test.** Repro the case where canonical
   `foo` and `foo_bar` both exist and a client supplies `foo_bar_xyz`;
   assert the selector returns `foo_bar`, not `foo`. This pins the
   shortest-qualifying-name tiebreaker against future regressions.
3. **Optional but worth it:** in the docstring's "wrong-collision" example,
   keep the existing `rain_gear` / `ear` example *and* add a
   `earthquake_alert` / `ear` example for the prefix-wrap direction. The
   rewrite mentions the latter in the docstring already (good) — would be
   nicer to mirror it as a `pytest.mark.parametrize` case.

## What I learned

The early-gate pattern — "only consider fuzzy matching when canonical has
the namespace separator" — is the right way to avoid a fuzzy matcher
silently taking over for unrelated user-defined tool names. Without that
gate, expanding suffix-wrap support would have introduced a subtle
substitution bug where a user function `scrape()` started matching the
proxy's `firecrawl-scrape` tool. The matcher rewrite is small; the
docstring's clear articulation of *why* the gate exists is what makes the
change safe to extend in this direction without regressing the original.
The PR-hygiene issue is independent: even a perfectly correct semantic
fix needs to be merge-able as a focused diff to land safely on a
fast-moving repo.
