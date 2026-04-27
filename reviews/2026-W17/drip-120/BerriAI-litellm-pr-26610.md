# PR #26610 — fix: increase health check max_tokens default to 16

- **Repo**: BerriAI/litellm
- **PR**: #26610
- **Head SHA**: `fd3eadbd`
- **Author**: hannahmadison
- **Size**: +8 / -9 across 2 files
- **URL**: https://github.com/BerriAI/litellm/pull/26610
- **Verdict**: **merge-as-is**

## Summary

Closes #23836. The health-check default for non-wildcard chat
models was `max_tokens: 5`, which is below the floor that several
modern providers accept on a chat completion (Anthropic's newer
models in particular reject very low caps with a 400, and a few
OpenAI-compat reasoning gateways pad reasoning tokens against the
cap before producing visible content, so 5 yields an empty
response that then trips the "no completion produced" branch and
flips the deployment to `unhealthy`). The fix bumps the default
from 5 → 16 in `_resolve_health_check_max_tokens` and updates the
matching pinning tests; the wildcard branch and the
explicit-override branches are untouched.

## Specific changes

- `litellm/proxy/health_check.py:319-353` —
  `_resolve_health_check_max_tokens` docstring step (5) and the
  return literal both move from `5` to `16`. The function's
  precedence chain (explicit per-model →
  `health_check_max_tokens_reasoning|non_reasoning` →
  `BACKGROUND_HEALTH_CHECK_MAX_TOKENS_REASONING` env →
  `BACKGROUND_HEALTH_CHECK_MAX_TOKENS` env → default 16 →
  wildcard `None`) is unchanged in shape; only the fallback
  literal moves.
- `tests/test_litellm/proxy/test_health_check_max_tokens.py:13-179` —
  three call sites updated:
  - `test_update_litellm_params_max_tokens_default` — assertion
    bumps from 5 → 16 (and docstring updated to match).
  - `test_update_litellm_params_max_tokens_wildcard` — tightens
    from `not in updated_params or updated_params["max_tokens"] != 1`
    to the cleaner `"max_tokens" not in updated_params` (this is
    a small but real fix: the previous assertion was structurally
    weaker and would have allowed a `max_tokens: 0` regression
    through).
  - `test_reasoning_specific_falls_through_when_wrong_branch_only`
    and `test_background_split_env_reasoning_vs_non_reasoning` —
    both updated to assert `== 16` on the fallthrough path.

## Risks

Behavior change is observable only on deployments that did not set
their own `health_check_max_tokens`. The user-visible effect is:
some health checks that were 400-failing or empty-completing on
`max_tokens: 5` will now succeed and report healthy. The reverse —
a deployment that was previously healthy at 5 becoming unhealthy
at 16 — would require a provider that *rejects* `max_tokens >= 16`
on chat, which is not a known shape.

Cost impact per health-check tick is trivial (16 output tokens at
the most expensive frontier model is fractions of a cent), and
health-check cadence is operator-controlled.

The wildcard-route fallthrough still returns `None` (caller omits
`max_tokens` entirely), preserving the existing "don't speak for
unknown deployments" invariant.

The companion PR #26604 (mentioned in the file map) appears to
strip `max_tokens` for non-chat handlers entirely, which is the
right complement — together they say "16 is the chat default,
and we don't pass it at all to image/embedding/rerank routes."

## Verdict rationale

Tiny, well-scoped, addresses a closed-with-evidence user issue.
Test updates are mechanical and the one substantive test
tightening (`!= 1` → `not in`) is a strict improvement. Merge as-is.
