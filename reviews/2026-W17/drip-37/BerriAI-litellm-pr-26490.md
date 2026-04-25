# BerriAI/litellm #26490 — Restrict /global/spend/* routes to admin roles

- **Repo**: BerriAI/litellm
- **PR**: [#26490](https://github.com/BerriAI/litellm/pull/26490)
- **Head SHA**: `01eee0944c837eaa017ec897b7031cfc2c367d46`
- **Author**: yuneng-berri
- **State**: OPEN (+137 / -7)
- **Verdict**: `merge-as-is`

## Context

`global_spend_tracking_routes` (e.g. `/global/spend/report`,
`/global/spend/teams`, `/global/spend/keys`,
`/global/spend/end_users`, `/global/spend/models`,
`/global/spend/provider`, `/global/spend/tags`,
`/global/spend/all_tag_names`, `/global/spend/logs`,
`/global/spend`) return spend aggregates *across every team,
customer, and api_key in the proxy*. They were nonetheless wired
into both `internal_user_routes` and
`internal_user_view_only_routes` in
`litellm/proxy/_types.py`, which meant a non-admin
`internal_user` key could call
`GET /global/spend/report?group_by=team` and walk away with the
full per-team spend ledger including hashed api_keys, models, and
token counts for every team in the proxy. Hard cross-tenant data
disclosure.

## Design

Two-line code change in `litellm/proxy/_types.py:651-672`:

- Drop `+ global_spend_tracking_routes` from
  `internal_user_routes` (the union now ends after
  `spend_tracking_routes + key_management_routes`).
- Replace `internal_user_view_only_routes = spend_tracking_routes
  + global_spend_tracking_routes` with `= spend_tracking_routes`.

A long block-comment is added above the union (lines 651-657)
documenting *why* the global routes are intentionally excluded —
"non-admin roles must not see other tenants' spend" — and
pointing to the two compensating paths:

1. `PROXY_ADMIN` and `PROXY_ADMIN_VIEW_ONLY` access flows through
   `_check_proxy_admin_viewer_access` in `route_checks.py`, which
   already allows `global_spend_tracking_routes`. PR description
   confirms `PROXY_ADMIN_VIEW_ONLY` to `/global/spend/{report,teams,keys,tags}` still 200s.
2. `permissions={"get_spend_routes": true}` keeps the explicit
   opt-in path open for admins who want to mint a non-admin key
   with global-spend access.

Tests are the impressive part:

- `tests/test_litellm/proxy/auth/test_route_checks.py` adds 4
  parametrized tests over the *full* `GLOBAL_SPEND_ROUTES` list
  (sourced from `LiteLLMRoutes.global_spend_tracking_routes.value`,
  not hardcoded) covering the four matrix cells:
  internal_user blocked, internal_user_view_only blocked,
  proxy_admin_view_only allowed, internal_user with explicit
  `get_spend_routes=true` permission allowed. 44 cases generated.
  Future additions to the global-spend route list are exercised
  *automatically* — that's the right shape.
- `tests/proxy_unit_tests/test_user_api_key_auth.py::test_ui_token_route_access`:
  flips `("/global/spend/logs", "internal_user", True)` →
  `False` and `("/global/spend/all_tag_names", "internal_user_viewer", True)` →
  `False`. The previous expected values *encoded the bug* — the
  test was passing because it asserted the leak, which is the
  classic "test-locked vulnerability" pattern.
- `tests/proxy_admin_ui_tests/test_role_based_access.py` does the
  same flip on `/global/spend/logs` for `INTERNAL_USER`.
- UI impact verified to be none —
  `ui/litellm-dashboard/src/components/usage.tsx` already gates
  the `global/spend/teams|keys|end_users` calls behind
  `isAdminOrAdminViewer(userRole)`. So the regression surface for
  the dashboard is empty.

## Risks

- **Customer rollout.** Anyone who has *already* been depending on
  this leak (intentionally exposing global spend to internal_user
  keys for cost-analytics dashboards) will see 401s after upgrade.
  PR doesn't have a feature-flag escape hatch. Reasonable —
  the right fix for that customer is to mint a key with
  `permissions={"get_spend_routes": true}` per the documented
  opt-in.
- **Test-flip integrity.** Three tests were flipping from `True`
  to `False`. That's exactly the right thing to do (a true→false
  flip *is* the regression test for the closed leak), but a
  reviewer should at minimum eyeball each one to confirm the
  *previous* assertion was the buggy expectation, not load-bearing
  for some other reason. The diff looks clean: each flip lines up
  with the role-route pair the PR is restricting.

## Suggestions

None blocking. Worth a CHANGELOG/release-note entry flagging this
as a **breaking** access-control tightening with the migration
guidance pointing to the `get_spend_routes` permission opt-in.

## What I learned

Auto-generating the test matrix from
`LiteLLMRoutes.global_spend_tracking_routes.value` rather than
hardcoding the route list is the right pattern for routes
declared via enum/list constants. It's the test-side of "single
source of truth" — a future addition that says
"oh and also `/global/spend/foo`" gets the matrix coverage for
free without anyone remembering to backfill the test.
