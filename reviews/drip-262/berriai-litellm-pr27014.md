---
pr: https://github.com/BerriAI/litellm/pull/27014
head_sha: 38fd1a99c4a948e81f6cd648d007c1f2d3817b4c
author: stuxf
additions: 459
deletions: 11
files_touched: 3
---

# Scope

Fixes two cross-tenant leaks in `/team/daily/activity` and
`/agent/daily/activity` (VERIA-43):

1. `/team/daily/activity` previously set `has_full_team_view = True` as
   soon as the caller had admin or activity permission for *any one*
   requested team — combining `?team_ids=A,B` with admin on A and
   member-only on B leaked B's per-API-key breakdown. Now the loop
   requires admin/permission on *every* requested team or falls back to
   the caller's own keys.
2. `/agent/daily/activity` ran `find_many` with an empty
   `where_condition` when `agent_ids` was omitted, returning every
   agent's spend rows on the proxy to any authenticated user with the
   `agents` feature. Now non-admin callers are scoped to permitted +
   created agents and an empty resolved set returns an empty page
   without a query.

Adds `tests/.../test_activity_tenant_scoping.py` (+385) with 6 tests.

# Specific findings

- `litellm/proxy/management_endpoints/team_endpoints.py:5028-5052`
  (head `38fd1a9`) — the loop now defaults `has_full_team_view = True`
  and breaks to `False` on the first team that lacks admin/permission.
  Correct logic for the "all-or-nothing" requirement. Two notes:
  - The fall-back behavior of "filter the *whole* response by the
    caller's own keys" is documented in the surrounding comment but is
    a real DX regression for callers who were previously seeing
    breakdowns for the team they admin. The PR body says "they can re-
    request the admin-only teams separately to get the wider view" —
    that's the correct trade-off given the leak, but consider returning
    an additional structured hint in the response indicating which
    teams were demoted, so client UIs can surface a "request these
    separately" tip rather than silently showing a narrower dataset.
  - The `team_aliases` fetch upstream in this handler is unchanged by
    this PR; if it ever returns fewer rows than `team_ids_list` (e.g.
    deleted teams), the loop's `for team_alias in team_aliases` won't
    flag the missing teams. Worth a defensive check that
    `len(team_aliases) == len(team_ids_list)` before declaring full
    view.
- `litellm/proxy/agent_endpoints/endpoints.py:977-1029` — the new
  scoping block:
  - Imports `AgentRequestHandler` and `_user_has_admin_view` *inside*
    the function. Stylistically consistent with the file but please
    confirm there's no circular-import reason for the deferral; if not,
    moving these to module top-level keeps cold-path latency tighter
    on the first request.
  - Lines 996-1003 explicitly guard against `user_id is None` so
    `find_many(where={"created_by": None})` never matches every
    ownerless row. Excellent — this is exactly the kind of footgun the
    Prisma `where=None` semantics invite, and the explanatory comment
    is well written.
  - Lines 1015-1033 return an empty `SpendAnalyticsPaginatedResponse`
    with all metadata zeroed when the resolved set is empty. Good — no
    DB hit. Confirm `DailySpendMetadata`'s field set still matches the
    constructor call (page/total_pages/has_more added) — the import on
    line 34 (in the diff context) suggests yes.
- `litellm/proxy/agent_endpoints/endpoints.py:1019-1030` —
  `agent_ids_list` is intersected with `permitted_agent_ids` *only*
  when `agent_ids_list` is non-empty; otherwise it is replaced. Correct
  behavior. Subtle: if a non-admin caller passes
  `agent_ids=foo,bar` where neither is permitted, they get the
  empty-page path — good, but the empty page reveals that some filter
  was supplied. Consider whether that is an information leak (probably
  not — the caller already knows which IDs they sent — but worth a
  thought).
- `tests/.../test_activity_tenant_scoping.py:53-105` — the
  `test_team_activity_requires_admin_on_every_requested_team` test
  asserts `captured["api_key"] == ["alice-key-1"]` after requesting
  team-A (admin) + team-B (member). Tight regression cover for the
  primary leak.
- `tests/.../test_activity_tenant_scoping.py:285-342` —
  `test_agent_activity_non_admin_no_perms_falls_back_to_owned`
  validates the owned-agents fallback. Mock structure is reasonable;
  confirm in CI that the two-call `side_effect=[owned, owned]` ordering
  matches the actual production call sequence.

# Suggested changes

1. Add a defensive `len(team_aliases) == len(team_ids_list)` check (or
   set comparison) before entering the all-teams loop to catch deleted
   teams.
2. Consider adding a structured hint in the response when the
   request is demoted to user-key filtering, so dashboards can surface
   a "split this request" suggestion instead of silently showing
   narrower data.
3. Move the local imports in `get_agent_daily_activity` to module
   top-level if no circular dependency exists.
4. Optional: add a unit test for the
   "non-admin sends agent_ids that are all unpermitted → empty page"
   path to lock the current behavior.

# Verdict

`merge-after-nits`

# Confidence

High — the bugs are clearly described, the fixes are minimal and
correct, and the tests cover both the regressions and the happy path.

# Time spent

~12 minutes.
