# BerriAI/litellm #26533 — fix(proxy): handle client-side unique-ID suffixes in MCP semantic tool filter

- **Repo**: BerriAI/litellm
- **PR**: #26533
- **Author**: Prithvi1994
- **Head SHA**: 76d8d2995127bc51ff4ec6aab41c13a4637a69ee
- **Link**: https://github.com/BerriAI/litellm/pull/26533
- **Size**: 56 lines added in
  `litellm/proxy/_experimental/mcp_server/semantic_tool_filter.py`,
  155 lines of new tests in
  `tests/test_litellm/proxy/_experimental/mcp_server/test_semantic_tool_filter.py`.

## What it changes

Adds a symmetric **suffix** match branch to
`SemanticMCPToolFilter._name_matches_canonical`
(`semantic_tool_filter.py:266-298`). Background: the prior fix in
#26117 handled the **prefix** case (clients like opencode wrap the
canonical with `<alias><sep><canonical>`). LibreChat does the
opposite — it appends a unique-ID suffix
(`<canonical><sep><uid>`) to disambiguate tools from multiple MCP
servers. The old code only checked `endswith(canonical)`, so
`fc_web_search-firecrawl_scrape_a1b2c3d4` matching canonical
`fc_web_search-firecrawl_scrape` returned `False`, the filter
dropped every tool, and the proxy forwarded `tools: []` with
`tool_choice: auto` — strict upstreams reject this with 400.

Key match logic at `:282-298`:

```python
if client_name.startswith(canonical):
    remainder = client_name[len(canonical):]
    if remainder and remainder[0] in ("_", "-"):
        rest = remainder[1:]
        if "_" not in rest and "-" not in rest:
            return True
```

The "no separator in `rest`" check is the anchor that prevents
`svc-search-extra_tool` from spuriously matching canonical
`svc-search` — without it, any longer namespaced tool starting with
the canonical prefix would collide.

## Strengths

- **The match anchor is symmetric and well-justified.** Comment at
  `:288-298` explicitly motivates the "no separator in remainder"
  rule with a concrete counterexample. Anyone maintaining this in
  the future has the failure mode written next to the code.
- **Both prefix and suffix paths are gated on `canonical` itself
  containing `MCP_TOOL_PREFIX_SEPARATOR`** (the early return at
  `:264-266` carried over from the prefix work). This keeps local
  user functions with no MCP namespace from getting swept in. The
  test `test_suffix_without_separator_in_canonical_does_not_match`
  (`test_semantic_tool_filter.py:725-741`) pins this.
- **Test coverage is exhaustive for the new branch:**
  `test_client_suffix_with_underscore_separator` (the LibreChat
  case), `test_client_suffix_with_dash_separator` (variant
  separator), `test_suffix_does_not_match_another_namespaced_tool`
  (the spurious-match guard), `test_exact_match_preferred_over_suffixed`
  (ordering stability), `test_prefix_and_suffix_both_match_same_canonical`
  (cross-client interaction), and a static-method test that pins
  the matcher contract directly. Six behavioral asserts plus one
  contract-level direct test for ~40 lines of production logic is
  the right ratio.
- **Returns the original incoming tool unchanged** (asserted in the
  first two suffix tests at `:670-697`) so the client-facing name
  survives for tool-call round-trips. Without that, the model would
  see "I called `fc_web_search-firecrawl_scrape`" but the response
  would map to `fc_web_search-firecrawl_scrape_a1b2c3d4` and break
  the round-trip.

## Concerns / asks

- **The "unique-ID has no separator" assumption holds for LibreChat
  today, but is an unstated contract.** LibreChat appears to use
  short hex (`a1b2c3d4`-style); other clients could append base64
  IDs containing `-`/`_`, or compound IDs like `client-instance_42`.
  Worth a docstring note at `:266-280` saying "if a future client
  uses separator-containing unique IDs, this matcher will reject
  them — tracked as a known limitation". Or, more rigorously: gate
  the suffix branch on a regex like `r"[A-Za-z0-9]+$"` so the
  contract is explicit in code, not just in the comment.
- **`test_prefix_and_suffix_both_match_same_canonical`
  (`:710-723`) asserts the prefix wins by way of "shortest
  qualifying name" — but the docstring says "shortest qualifying"
  while the actual `_get_tools_by_names` selection logic isn't
  shown in this diff. If the selection is "first match in
  `available_tools` order", a tools list ordered `[suffixed,
  prefixed]` would fail this test. Worth asserting against the
  selection contract directly (e.g. by inspecting the
  `_get_tools_by_names` body or pinning both orderings).
- **No test for the `client_name == canonical` exact-match
  short-circuit** (`:262`). Trivial but worth one line so a future
  refactor can't silently delete the early-out.
- **Consider profiling the cost.** For a 200-tool list × 200
  canonical names, this is 40k `startswith`/`endswith` calls per
  request. Probably fine, but a one-line comment noting the O(n²)
  expectation, or a TODO to memoize on the (canonical-set, tool-set)
  pair, would help.

## Verdict

**merge-after-nits** — fix is correct, well-tested, and
well-documented. The asks are: (1) document the "unique-ID
separator-free" assumption either as a regex or a comment; (2) one
extra test for the exact-match path. Both are 5-line additions
and shouldn't gate the merge.

## What I learned

When a filter has both a prefix and a suffix wrapping convention
in the wild, the symmetric anchor is the same idea twice but the
*spurious-match guard* is asymmetric: prefix anchors on "preceded
by separator", suffix anchors on "followed by separator AND
remainder contains no further separator". The asymmetry is because
namespacing prefixes are length-bounded by convention, but
unique-ID suffixes are length-bounded by *absence of separator*.
Both are fragile contracts; both deserve a regex.
