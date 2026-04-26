---
pr: 26533
repo: BerriAI/litellm
sha: 76d8d2995127bc51ff4ec6aab41c13a4637a69ee
verdict: merge-after-nits
date: 2026-04-26
---

# BerriAI/litellm #26533 — fix(proxy): handle client-side unique-ID suffixes in MCP semantic tool filter

- **URL**: https://github.com/BerriAI/litellm/pull/26533
- **Author**: Prithvi1994 (Prithvi Monangi)
- **Head SHA**: 76d8d2995127bc51ff4ec6aab41c13a4637a69ee
- **Size**: +216/-18 across 2 files (`litellm/proxy/_experimental/mcp_server/semantic_tool_filter.py` + `tests/.../test_semantic_tool_filter.py`)
- **Fixes**: #26507

## Scope

Symmetric extension of `SemanticMCPToolFilter._name_matches_canonical` at `semantic_tool_filter.py:221-302` to recognize the **suffix** form `<canonical><sep><unique_id>` (LibreChat pattern) in addition to the existing **prefix** form `<alias><sep><canonical>` (added in #26117 for opencode). Without this, LibreChat tool names like `fc_web_search-firecrawl_scrape_a1b2c3d4` were dropped by the filter, the proxy forwarded `tools: []` with `tool_choice: auto`, and strict upstream providers returned 400.

7 new unit tests in `TestGetToolsByNames` cover: underscore-suffix, dash-suffix, false-positive guard against another namespaced tool, unprefixed-canonical safety, exact-match-preferred ordering, prefix+suffix coexistence, and direct static-method verification.

## Specific findings

- **The core algorithmic guard is sound.** `semantic_tool_filter.py:280-296` accepts a suffix match only if the remainder after the canonical+separator contains *neither* `_` nor `-`. This rejects `svc-search-extra_tool` against canonical `svc-search` — the `extra_tool` segment contains both, so it's correctly identified as another namespaced tool rather than a unique-ID. The choice to reject *both* separators (rather than just `MCP_TOOL_PREFIX_SEPARATOR`) is conservatively correct because clients don't know the proxy's separator (same reasoning as the original prefix branch).

- **`MCP_TOOL_PREFIX_SEPARATOR in canonical` gating is preserved at `:264`** for both branches. This is the right safety net: an unprefixed canonical like `firecrawl_scrape` doesn't trigger suffix matching against `my_firecrawl_scrape` (which would otherwise spuriously match because `firecrawl_scrape` is a complete prefix of `my_firecrawl_scrape` followed by `_`). `test_suffix_without_separator_in_canonical_does_not_match` at line ~755 covers this.

- **`test_prefix_and_suffix_both_match_same_canonical` at lines ~775-790 has a comment that's slightly aspirational** ("Should match the shortest qualifying name (prefix case)"). The test asserts `len(matched) == 1` and `matched[0]["name"] == prefixed`. But the actual selection logic isn't visible in the diff — `_get_tools_by_names` apparently iterates `available_tools` in registration order and stops at the first match, so what really wins is *insertion order*, not *shortest name*. The test happens to pass because the test fixture inserts `prefixed` first. Worth either (a) reordering the fixture to assert order-independence, or (b) tightening the comment to "first match wins; relies on insertion order", or (c) adding actual length-tiebreak logic in `_get_tools_by_names`. (a) or (b) is the merge-after-nits ask; (c) is a follow-up PR.

- **Unique-ID character set assumption is undocumented.** The comment at `:288` says "A unique-ID segment contains no separator (it's a short hex or alphanumeric string)". Real LibreChat suffixes are 8-char hex (`a1b2c3d4`), but I can't see anywhere in the proxy that *enforces* that. If a future MCP client uses kebab-case nanoids (`abc-def`) or snake-cased ULIDs, the filter will silently drop them again with the same symptom (tools: []). Worth surfacing a one-line `verbose_proxy_logger.debug` when `client_name.startswith(canonical)` but the remainder is rejected — it would have made #26507 self-diagnosing rather than requiring a roundtrip with the issue reporter.

- **Length guard is correctly placed.** `:265` `if len(client_name) <= len(canonical): return False` runs before either branch, so `client_name == canonical` is handled by the early-return at `:262` and the suffix branch never sees a zero-length remainder. Defensive.

- **No change to `_get_tools_by_names`, `filter_tools`, or any other method.** The PR description's "purely additive — a new `if` branch after the existing prefix check" claim is accurate. Risk to existing prefix-clients (opencode users) is therefore zero — `if client_name.endswith(canonical)` still runs first and short-circuits via the `return True` on success.

- **Author drive-by quality is high.** Issue-linked, root-cause-identified, table-of-symptoms in the PR body, 7 tests with named scenarios, no churn beyond the diff. Maintainer-friendly.

## Risk

Low. The new branch only fires when (a) prefix match failed, (b) `client_name.startswith(canonical)` is true, (c) the canonical itself contains `MCP_TOOL_PREFIX_SEPARATOR`, and (d) the remainder is single-segment. Any existing flow that wasn't dropping tools is unaffected; any flow that was dropping LibreChat-style suffix tools now gets them through. Worst case: a future client uses a unique-ID format containing a separator and continues to be dropped — but that's the same behavior as pre-PR (no regression).

## Nits

1. Tighten or fix the order-dependence in `test_prefix_and_suffix_both_match_same_canonical` (`test_semantic_tool_filter.py:~775-790`).
2. Add a `verbose_proxy_logger.debug` when `startswith(canonical)` but the remainder is rejected, so future "tools: [] with no obvious cause" reports are self-diagnosing.
3. Consider documenting (in the docstring at `:221-258`) the unique-ID character-set assumption: "alphanumeric only; clients using separator-containing IDs are not supported".

## Verdict

**merge-after-nits** — correct fix for a real production-blocking bug, conservative extension of the existing matcher with the right safety guards. The three nits above are observability and test-clarity improvements, not correctness blockers.

## What I learned

When a matching algorithm sits between two systems with independent naming conventions (canonical-side vs. client-side), every "client wraps the canonical" pattern needs a *symmetric* matcher: `endswith(canonical)` + `startswith(canonical)`. The danger isn't getting one of them wrong — it's getting one of them implemented and forgetting the other, then tripping over it months later when a new client picks the missing pattern. The diagnostic signal in #26507 ("tools: [] with tool_choice: auto, then 400") is also generalizable: any filter that silently produces an empty allowlist needs a debug-log breadcrumb at the point of rejection so the next client compatibility issue takes minutes to root-cause instead of days.
