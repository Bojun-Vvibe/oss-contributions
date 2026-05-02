# BerriAI/litellm PR #27053 — fix(anthropic,vertex): three reasoning_effort bugs

- URL: https://github.com/BerriAI/litellm/pull/27053
- Head SHA: `d0543060ccfb1847f51a04bf316f0d4db91d535b`
- Files touched: 6 (+178 / −53)
- Verdict: **merge-after-nits**

## Summary
Follow-up to PR #27039's QA matrix. Tightens `reasoning_effort` handling on the
Anthropic / Vertex transformations: (a) introduces a canonical
`_VALID_REASONING_EFFORT_VALUES` frozenset, (b) validates up front so invalid
strings fail consistently across model generations instead of being silently
accepted as `{type: "adaptive"}` on Claude 4.6/4.7, and (c) drops the now-unused
`DEFAULT_REASONING_EFFORT_MINIMAL_THINKING_BUDGET` import.

## Cited concerns

1. `litellm/llms/anthropic/chat/transformation.py:802-812` (post-patch):
   `_VALID_REASONING_EFFORT_VALUES = frozenset({"none","minimal","low","medium",
   "high","xhigh","max"})`. This duplicates the `REASONING_EFFORT` Literal/enum
   used elsewhere in the codebase. Extracting into a shared module
   (`litellm/types/llms/openai.py` or wherever `REASONING_EFFORT` lives) and
   sourcing the frozenset from `typing.get_args(REASONING_EFFORT)` would
   prevent drift the next time someone adds an effort tier.

2. `litellm/llms/anthropic/chat/transformation.py:819-833` (post-patch): the
   new validation block raises `ValueError`, with the comment noting that
   PR #27050 will convert it to `litellm.BadRequestError`. That cross-PR
   coupling is fragile — if #27050 lands first or is abandoned, callers
   continue to see opaque 500s. Either land both together or convert here
   to `litellm.BadRequestError(status_code=400, …)` directly and let #27050
   no-op the duplicated logic.

3. The diff removes the `DEFAULT_REASONING_EFFORT_MINIMAL_THINKING_BUDGET`
   import. Confirm no other code path in this file still relies on
   `minimal` mapping to a budget — if so, the constant should remain even if
   this particular function no longer needs it. A `rg` for the symbol across
   the repo will tell quickly.

4. PR description references "three bugs" but the visible diff is mostly the
   validator + import cleanup. A bullet list of which test in the new tests
   covers which bug would help reviewers map fix→regression test 1:1.

## Verdict
**merge-after-nits** — correctly converts a silent-misbehavior class into a
loud failure. Fix the cross-PR error-type coupling, dedupe the effort-set with
the Literal type, and ship.
