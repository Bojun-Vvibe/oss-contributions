# litellm #26499 — join team-member budget so rpm/tpm limits are enforced

- URL: https://github.com/BerriAI/litellm/pull/26499
- Head SHA: `9f1e970a68d7fe5e0f7f3b84fb6dc4aec32771a7`
- Verdict: **merge-as-is**

## What it does

Fixes #26378. The team-member RPM/TPM limit attached to a virtual key was
never enforced because `combined_view` in `litellm/proxy/utils.py` joined
`LiteLLM_TeamMembership` but only pulled `tm.spend AS team_member_spend` —
it never reached over to `LiteLLM_BudgetTable` via `tm.budget_id`. The
auth path read `team_member_*_limit` columns that were always `NULL`, so
the rate-limiter silently fell through to "no limit".

## Diff notes

- Schema change is one extra `LEFT JOIN`:
  ```
  LEFT JOIN "LiteLLM_TeamMembership" AS tm
    ON v.team_id = tm.team_id AND tm.user_id = v.user_id
  + LEFT JOIN "LiteLLM_BudgetTable" AS tmb ON tm.budget_id = tmb.budget_id
  ```
  And new selected columns mapped onto `team_member_rpm_limit` /
  `team_member_tpm_limit` / `team_member_max_budget` /
  `team_member_budget_duration`.
- `LEFT JOIN` is the right choice — keys with no team membership or no
  per-member budget should still resolve, just with `NULL` limits (which the
  enforcer correctly treats as "no per-member cap").
- The new regression test
  `test_view_carries_team_member_rate_limits_from_combined_view_row`
  constructs the view from the same kwargs the SQL row delivers, which is
  the exact shape the production code path consumes. That's a good way to
  pin the column-mapping contract without standing up a real DB.
- `model_prices_and_context_window_backup.json` carries unrelated Azure
  GPT-5.5 pricing entries. Not a behavioral concern but should be split in
  an ideal world.

## Risk surface

- The new join is on `(team_id, user_id)` for membership and then `budget_id`
  for the budget row. If a team-member row exists with a stale `budget_id`
  pointing at a deleted budget, the LEFT JOIN gives NULL columns and the
  enforcer falls back to "no limit" — same behavior as before the fix, so
  no regression risk introduced here.
- One tiny worry: `combined_view` is a hot path. Adding a join on a small
  table (`LiteLLM_BudgetTable`) is cheap, but if `tm.budget_id` is not
  indexed on the budget table, large deployments may want to verify the
  query plan. Worth a perf note in the PR description.

## Why this verdict

Correct root-cause fix for a silent security/billing bug, minimal blast
radius, regression test pins the exact column contract. Pricing-JSON noise
is the only concern and is easily squashed at merge time.
