# BerriAI/litellm PR #26498 — fix(auth): apply temp_budget_increase on cache-hit path

- **URL:** https://github.com/BerriAI/litellm/pull/26498
- **Head SHA:** `46509bf2e9d065b1301484a392aa80312eddb598`
- **Files touched:** 7 (auth fix + caching test, plus a noisy 620-line `model_prices_*.json` add for unrelated azure/gpt-5.5 entries)
- **Verdict:** `merge-after-nits`

## Summary

Closes #25760. `_update_key_budget_with_temp_budget_increase` was only
called inside the DB-fetch branch of `_user_api_key_auth_builder`, so
when a token came back from the in-memory or Redis cache the temp
budget bump was never applied — across replicas a key bumped from
`max_budget=2.0` to `102.0` would intermittently still enforce `2.0`.
Fix moves the call to the convergence point and switches the helper to
return a `model_copy` so cached references aren't mutated.

## Specific references

- `litellm/proxy/auth/user_api_key_auth.py:1221-1230` — call hoisted
  out of the DB-fetch block to a single convergence point guarded by
  `if valid_token is not None`. This is the actual bug fix.
- `litellm/proxy/auth/user_api_key_auth.py:1722-1735` — helper now
  early-returns when `temp_budget_increase == 0.0` and otherwise
  returns `valid_token.model_copy(...)` instead of mutating the cached
  pydantic instance. Without this, fixing the cache-hit path would
  compound the bump on every request.
- `tests/test_litellm/proxy/auth/test_temp_budget_increase_caching.py`
  (new, 232 lines) — covers no-mutation, repeated calls don't compound,
  identity preservation when no increase, expiry, `max_budget is None`,
  and the original cache-hit regression.

## Reasoning

Root cause analysis is correct, the immutability fix is the right
shape (compounding would have been the obvious follow-up bug), and the
test file directly exercises both the no-mutation invariant and the
cache-hit regression. The `valid_token is not None` guard is necessary
because the new call site is reached even on the early-bail paths.

Nit: the PR body claims it's a focused auth fix but bundles
`model_prices_and_context_window{,_backup}.json` adds for
`azure/gpt-5.5*` (310 lines × 2). Those should be a separate commit/PR
— they're unrelated to the auth bug and pollute the diff for reviewers
and `git blame`. Ask the author to split before merge.
