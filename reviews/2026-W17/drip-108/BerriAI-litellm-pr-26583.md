# BerriAI/litellm #26583 — [Feature] Allow Admin Viewers to Access Spend Logs

- **Repo**: BerriAI/litellm
- **PR**: #26583
- **Author**: kimsehwan96
- **Head SHA**: 1c8901ff710ce147134b36ebe977ae3d8d09d594
- **Size**: +106 / −0 across two files (one prod schema, one test).

## What it changes

Two coupled edits to `litellm/proxy/_types.py` `LiteLLMRoutes`
enum:

1. **Adds two missing routes to `spend_tracking_routes`** at
   `_types.py:586-590`:
   - `/spend/logs/v2`
   - `/spend/logs/ui/{request_id}`

   The first is the v2 paginated logs endpoint that
   already existed in code but was unreachable for
   non-proxy-admin roles because the route check enum
   never knew about it. The second is the per-request
   detail view that the admin UI already linked to from
   the Logs page but that 403'd for any role except full
   `PROXY_ADMIN`.

2. **Adds the same five `/spend/logs/*` routes to
   `proxy_admin_view_only_routes`** at `_types.py:729-733`:
   - `/spend/logs`, `/spend/logs/ui`, `/spend/logs/v2`,
     `/spend/logs/session/ui`, `/spend/logs/ui/{request_id}`.

   This is the load-bearing change: the
   `PROXY_ADMIN_VIEW_ONLY` role exists to "view all spend
   across the platform" (per its own role description) but
   was previously locked out of the raw spend log endpoints
   that back the admin Logs UI page — so a viewer could
   load the dashboard but every grid cell rendered as a
   permission error.

## Strengths

- **Right scope.** This is a route-enum gap fix, not a new
  authorization model. The endpoint handlers themselves
  already enforce per-user data filtering via
  `_can_user_view_spend_log` (called out in the second test
  docstring at `:1290-1297`), so widening the route enum
  for the two viewer-class roles doesn't expand
  data-visibility — it just lets the route check succeed
  before the per-row filter takes over.
- **Excellent test coverage discipline** at
  `tests/test_litellm/proxy/auth/test_route_checks.py:1198-1297`:
  - First test (`test_proxy_admin_viewer_can_access_spend_logs`)
    parametrizes all five routes and asserts that
    `non_proxy_admin_allowed_routes_check` does not raise
    for `PROXY_ADMIN_VIEW_ONLY`. Catches future enum
    drift (someone deleting a route from
    `proxy_admin_view_only_routes` would fail the
    matching parametrize case).
  - Second test (`test_internal_user_can_access_v2_spend_logs`)
    parametrizes both `INTERNAL_USER` and
    `INTERNAL_USER_VIEW_ONLY` against
    `/spend/logs/v2` and `/spend/logs/ui/{request_id}`,
    pinning the parity-with-`/spend/logs/ui` argument the
    PR makes (these two routes share their handler with
    the already-allowed `/spend/logs/ui` so they must be
    reachable by the same roles).
  - Both tests use `pytest.fail(f"... should be able to
    access {route} ...")` rather than `assert ... is None`,
    which gives operators a self-explaining failure
    message instead of "AssertionError: assert None is
    None" when the route lookup regresses.
- **Path-template correctness.** `/spend/logs/ui/{request_id}`
  is added with the curly-brace placeholder rather than as
  a regex or wildcard, matching the convention already in
  use for `/audit/{id}` at `:728`. The route-check matcher
  (which the test exercises with the literal value
  `/spend/logs/ui/some-request-id`) clearly handles
  template-to-literal matching, so the contract is pinned
  end-to-end.

## Concerns / nits

- **No negative test.** The PR adds five routes to
  `proxy_admin_view_only_routes` and exercises that they
  pass the check, but doesn't pin that
  `INTERNAL_USER_VIEW_ONLY` *cannot* access
  `/spend/logs/session/ui` (which is in
  `spend_tracking_routes` but is a UI endpoint that
  internal-user-view-only might not be intended to see).
  A negative-case test for at least one role/route pair
  would lock the boundary.
- **The duplication between `spend_tracking_routes` and
  `proxy_admin_view_only_routes`** is now five lines of
  copy-pasted strings. Not a blocker — that's the
  established pattern in the file — but a future
  refactor consolidating these into named constants
  (`SPEND_LOGS_ROUTES = [...]`) referenced from both
  enums would prevent drift.
- **PR title says "Spend Logs"** but the diff also touches
  `/spend/logs/v2` for `PROXY_ADMIN_VIEW_ONLY`'s
  `spend_tracking_routes` membership (line 588) — that's
  a separate v1-vs-v2-API parity question. Worth one
  PR-body sentence confirming the v2 endpoint is intended
  to expose the same row schema as v1 (otherwise
  permission semantics could subtly differ).

## Verdict

**merge-as-is.** Schema-level, route-enum gap fix with
parametrized positive tests covering both affected
roles and all five routes, well-anchored to the
existing role-description contract. The negative-test
and dedupe items are nice-to-haves for a follow-up.

## What I learned

When an authorization model has two layers — "can this
role hit this URL pattern at all?" and "can this user
see this row?" — gaps in the first layer present as
"the dashboard loads but every cell is a permission
error", which users diagnose as a UI bug rather than
an auth-config gap. The fix is mechanical (add the
route to the role's allowlist) but the diagnosis is
hard because the route-check failure happens *before*
the per-row filter even runs, so the row-level audit
trail is empty. Worth keeping a "missing route in role
enum" suspicion early in the differential when a
viewer-role user reports "everything is forbidden".
