# BerriAI/litellm #26659 — fix(projects): non-admin users can now see their projects

- URL: https://github.com/BerriAI/litellm/pull/26659
- Head SHA: `7be66dda1a4b629a85815863dbfa742f4a8d800d`
- Files: 4 (project_endpoints.py, _types.py, useProjects.ts, test file)
- Size: +99 / −22

## Summary

Closes a real "non-admin sees empty projects list" bug rooted in a
schema-vs-code mismatch: team membership is written to
`LiteLLM_UserTable.teams` by the `team_member_add` flow, but
`/project/info` and `/project/list` were querying
`LiteLLM_TeamTable.members`/`.admins` — columns that the membership-write
path *never populates*. So every non-admin user got `is_team_member = False`
and `team_ids = []`, regardless of how many teams they actually belonged
to. The fix queries the user-table side instead, and registers both
project routes as `self_managed_routes` so the framework stops 401-ing
them before they reach the membership check.

## Specific references

- `project_endpoints.py:855-862` rewrites `project_info` to look up
  `litellm_usertable.find_unique({"user_id": user_api_key_dict.user_id})`
  and check `project.team_id in user_row.teams`. The two-line code
  comment naming the data-flow asymmetry ("Membership is written to
  LiteLLM_UserTable.teams by team_member_add. team.admins and team.members
  are never populated by that path.") is exactly the kind of comment
  this class of bug deserves — it tells the next reader *why* the
  obvious code shape is wrong.
- `project_endpoints.py:911-920` mirrors the same fix in `list_projects`.
  The `if user_api_key_dict.user_id:` guard at `:914` is correct (the
  prior team-table-OR query would have returned `[]` for null user_id
  anyway, so this preserves behavior). The `user_row.teams if (user_row and user_row.teams) else []`
  pattern correctly handles three null cases (no row, row with null
  teams, row with empty teams).
- `_types.py:697-700` adds `/project/list` and `/project/info` to
  `LiteLLMRoutes.self_managed_routes` — load-bearing because without
  this registration, a non-admin caller is rejected at the framework
  routing layer with 401 before the per-endpoint membership check ever
  runs. The trailing comment "endpoint scopes results to caller's teams
  (non-admin)" is consistent with the existing
  `/guardrails/submissions` entry above it.
- `useProjects.ts:78-82` removes the client-side
  `all_admin_roles.includes(userRole || "")` gate, so the dashboard
  actually fires the request for non-admins instead of suppressing it.
  This is the *visible* half of the fix — without it, the backend fix
  alone wouldn't change what the UI shows.
- Test at `test_project_endpoints_prisma.py:868-878`
  (`test_project_routes_in_self_managed_routes`) is a static-config
  pin that asserts both routes are in the list — cheap, fast, and
  prevents a future "let's clean up self_managed_routes" sweep from
  silently re-breaking this. Test at `:881-947`
  (`test_list_projects_internal_user_queries_user_table_teams`) is
  the behavior pin: mocks `litellm_usertable.find_unique` to return
  `fake_user.teams = [team_id]` and *asserts*
  `mock_prisma.db.litellm_teamtable.find_many.assert_not_called()` —
  which is the strongest possible regression protection because a
  future "let's also fall back to team-table for safety" change would
  fail this test loudly.

## Risk

Low. The new query path strictly returns a *superset* of what the old
broken path returned (the old path returned `[]` for non-admins; the
new path returns whatever the user-table actually has). For the small
historical edge case where `team.members`/`.admins` *was* populated
(e.g., directly-mutated rows from a custom admin script), the new code
will now miss those teams — but the PR author's data-flow analysis
implies this never happened through any standard write path, so the
practical regression surface is empty.

## Nits (non-blocking)

- The two `find_unique` queries on `litellm_usertable` in `project_info`
  and `list_projects` could be a single helper
  `_get_user_team_ids(prisma_client, user_id) -> list[str]` to prevent
  drift if a future change adds e.g. organization-scoped team lookups.
- No test for the `project_info` fix specifically — only `list_projects`
  is covered. A symmetrical test mocking `litellm_projecttable.find_unique`
  + `litellm_usertable.find_unique` and asserting `is_team_member = True`
  for the membership-via-user-row case would close coverage.
- The migration story for any deployment that *did* successfully populate
  `LiteLLM_TeamTable.members/admins` (perhaps via a custom script or a
  prior version of the codebase) isn't mentioned. A one-line note in the
  PR ("if your DB has populated team.members/admins, those entries are
  now ignored — re-add via team_member_add") would help operators audit.
- `is_admin = user_api_key_dict.user_role in [LitellmUserRoles.PROXY_ADMIN, ...]`
  isn't visible in the diff (presumably already there above `:855`),
  but the `not (is_admin or is_team_member)` 403 path at `:864` should
  carry the user_id and team_id in the error message for support
  diagnostics — the current "not authorized" message is hard to triage.

## Verdict

`merge-as-is` — textbook "the schema and the code disagree" fix with
the right data-flow comment, the right behavior-pinning test
(`teamtable.find_many.assert_not_called()`), and the right three-piece
shape (backend query fix + route registration + UI gate removal).
