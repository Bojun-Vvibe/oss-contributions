# BerriAI/litellm #26498 — fix(auth): apply temp_budget_increase on cache-hit path

- URL: https://github.com/BerriAI/litellm/pull/26498
- Head SHA: `46509bf2e9d065b1301484a392aa80312eddb598`
- Verdict: **merge-after-nits**

## What it does

Closes #25760 — `temp_budget_increase` was applied only on the DB-fetch
branch of `_user_api_key_auth_builder` in
`litellm/proxy/auth/user_api_key_auth.py`, so the cache-only path
(`get_key_object(check_cache_only=True)`) returned a token that still
enforced the stale base `max_budget`. On Redis-backed multi-replica
deployments, replicas that didn't run the original DB fetch
intermittently saw e.g. `Max budget: 2.0` instead of `102.0`.

## Diff notes

Two coordinated changes in `litellm/proxy/auth/user_api_key_auth.py`:

1. `_update_key_budget_with_temp_budget_increase` is reworked to
   return a `valid_token.model_copy(...)` instead of mutating the input
   in place. Critical: the in-memory cache stores object references, so
   simply moving the original (in-place) call onto the cache path
   would have compounded the increase across requests. Returning the
   same instance unchanged when there's nothing to do (no temp
   increase, expired, `max_budget is None`) preserves identity for the
   no-op path.
2. The call moves to the convergence point right before budget
   enforcement, so both DB-fetch and cache-hit paths apply it exactly
   once.

The new test file
`tests/test_litellm/proxy/auth/test_temp_budget_increase_caching.py`
(232 lines) covers the right invariants:
- helper does not mutate input
- repeated calls do not compound (3× returns base + temp, not base + 3×temp)
- returns same instance when no temp increase / when expired / when
  `max_budget is None`
- end-to-end cache-hit path through `_user_api_key_auth_builder`
  applies the increase and leaves the cached reference at base.

## Nits

- The PR also rolls in 310-line additions to both
  `model_prices_and_context_window.json` and its `_backup.json`
  counterpart, plus a `test_llm_cost_calc_utils.py` change. These are
  unrelated catalog updates piggybacking on an auth fix — would much
  rather see the auth fix isolated and the price catalog as its own
  PR. Not a blocker if the maintainers usually batch like this, but
  flagging.
- `model_copy` without `deep=True` is fine here because `max_budget`
  is a scalar and the only mutated field, but a comment to that effect
  would prevent a future refactor from silently sharing nested state.

## Why this verdict

Real bug, correct fix shape (immutable copy + single convergent
application point), and the regression tests target the actual
multi-replica failure mode. Just want the price-catalog churn split
out.
