# BerriAI/litellm #27021 — [Fix] Proxy: Skip Personal Budget Hook When Reservation Covers Counter

- Link: https://github.com/BerriAI/litellm/pull/27021
- Head SHA: (latest commit on PR; merged into main)
- State: MERGED, +224/-1 across multiple files

## Summary
Fixes a boundary regression introduced by the budget-reservation feature
(PR #26845). The reservation path admits requests at strict-`<` and
atomically pre-fills the same `spend:user:{user_id}` counter that the
legacy `_PROXY_MaxBudgetLimiter.async_pre_call_hook` reads with `>=`.
When a fresh user submits a request with no `max_tokens` cap, the
reservation falls back to reserving the smallest remaining headroom and
fills the counter to exactly `max_budget` — and then the legacy hook
immediately rejects the request the reservation just admitted with a 429
"Max budget limit reached." This PR makes the legacy hook short-circuit
when the request's `budget_reservation` already covers the user counter.

## Specific references
- `litellm/proxy/hooks/max_budget_limiter.py:35-49` — new lazy import of
  `get_reserved_counter_keys` from
  `litellm.proxy.spend_tracking.budget_reservation`, then
  `if user_counter_key in get_reserved_counter_keys(...)` early `return`.
  Comment is excellent and pins the invariant.
- `tests/test_litellm/proxy/hooks/test_max_budget_limiter.py` — new file
  (208 lines) with three focused tests:
  - `test_under_budget_passes` — baseline OK path.
  - `test_over_budget_rejects_without_reservation` — confirms 429 still
    fires when no reservation is present.
  - `test_skips_when_user_counter_is_reserved` — pins the regression: a
    reservation entry for `spend:user:user-1` causes the hook to return
    without ever consulting `get_current_spend`. The assertion verifies
    `mock_get_spend` was *not* called.

## Notes
- The lazy import is appropriate — `proxy.utils` would otherwise create
  a circular import via `proxy_server`. The comment explicitly calls this
  out.
- The fix is narrow: it only skips when the *user* counter is reserved.
  Team / org budget hooks (separate code paths) are untouched.
- The negative test (`mock_get_spend.assert_not_called()` is implied by
  the `return` semantics) is the right way to prove the hook bailed
  out before the racy read.

## Verdict
`merge-as-is`
