---
pr: BerriAI/litellm#26809
sha: 04687ba48e706d4d09a578c941fcd1a72b51c0e9
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# [Fix] Team member null budget fallback

URL: https://github.com/BerriAI/litellm/pull/26809
Files: `litellm/proxy/auth/auth_checks.py`,
`tests/test_litellm/proxy/auth/test_auth_checks.py`
Diff: 132+/0-

## Context

`_check_team_member_budget` at `auth_checks.py:3293-3299` was treating
the existence of a `team_membership` row with a non-null
`litellm_budget_table` join as sufficient to extract `max_budget`.
But the budget table itself can have `max_budget = None` (a row
created by cloning from an incomplete team-default budget), and the
code then proceeded to use `None` as the budget cap downstream — at
best skipping enforcement, at worst mis-comparing in the budget
arithmetic. The fix is the obvious missing third condition:
`team_membership.litellm_budget_table.max_budget is not None`.

## What's good

- Three-condition `if` at `:3293-3296` is the textbook null-clone
  fallthrough fix:
  ```
  team_membership is not None
  and team_membership.litellm_budget_table is not None
  and team_membership.litellm_budget_table.max_budget is not None
  ```
  Single added line; no other code path touched. Falls through to
  the existing `else` branch which already handles the
  "fetch the team-default cap from the budget table" path.
- New test
  `test_team_member_budget_check_null_clone_falls_back_to_team_default`
  at `test_auth_checks.py:2342-2389` builds the exact corruption
  scenario: per-member row exists, joins to a budget row, but that
  budget row has `max_budget=None`. Asserts that the team-default
  `max_budget=65.0` from the fallback Prisma fetch is applied and
  the spend (`500.0`) trips
  `BudgetExceededError(current=500.0, max=65.0)`.
  `prisma_client.db.litellm_budgettable.find_unique.assert_awaited_once()`
  verifies the fallback path actually fired (not just that the
  existing per-member-budget path coincidentally did the right
  thing).
- Companion test
  `test_team_member_budget_check_null_clone_with_null_default_skips_enforcement`
  at `:2426-2486` covers the "both null" case — when even the
  team-default has `max_budget=None`, enforcement correctly skips
  rather than raising on `None` arithmetic.
- Both tests set up the same `LiteLLM_TeamMembership` /
  `LiteLLM_BudgetTable` fixtures with explicit `max_budget=None`
  and patch `get_team_membership` + `get_current_spend` — clean
  isolation, no cross-test state.

## Nits

- None blocking. Optional: a `LiteLLM_BudgetTable` model invariant
  that rejects creation with `max_budget=None` *unless* explicitly
  flagged as `is_template=True` would prevent this null-clone class
  at the schema level rather than the consumer level. But that's a
  schema migration outside this PR's scope.
- Consider auditing other consumers of `team_membership.litellm_budget_table`
  for the same "row exists but field is null" gap class. A grep
  for `litellm_budget_table` in `litellm/proxy/` would surface
  parallel consumers (key-budget checks, spend-window resets) that
  may have the same shape.
- The `assert_awaited_once()` is good; consider also asserting
  `find_unique.call_args` matches `where={"budget_id":
  "budget-default"}` to fence against a future refactor that fetches
  the wrong budget row.

## Verdict reasoning

One-condition fix to a real null-arithmetic / silent-skip-enforcement
gap, with two tests that lock both the fallback-applies and
fallback-also-null branches at exactly the right granularity. Zero
collateral risk: the only behaviour change is that previously-broken
clones now correctly fall through to the team-default budget. The
test setup names the exact corruption it's reproducing
("the cloned-from-incomplete-default case") which is excellent
documentation. Landable as-is.
