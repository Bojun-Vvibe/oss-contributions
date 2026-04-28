# BerriAI/litellm #26664 — fix(projects): project dropdown empty for internal_user (3 bugs)

- URL: https://github.com/BerriAI/litellm/pull/26664
- Head SHA: `7c1ffaf0994bf2121c5ea845616e476cf5bafa27`
- Size: +25 / -19 across 3 files
- Verdict: **merge-after-nits**

## What the change does

Diagnoses and fixes three independent bugs that all had to be wrong for the
"create virtual key" Project dropdown to be empty for `internal_user`. Each is
a real bug, not just a workaround:

1. **Frontend gate too strict**
   `ui/litellm-dashboard/src/app/(dashboard)/hooks/projects/useProjects.ts:78-81`:
   `enabled: Boolean(accessToken) && all_admin_roles.includes(userRole || "")`
   meant the React-Query hook never even **fired** for non-admins. Fix flips
   to `enabled: Boolean(accessToken)` and drops the unused
   `all_admin_roles` import. Backend remains the authority on what each
   user can see — the frontend should not pre-filter.

2. **Auth-routes table missing entries**
   `litellm/proxy/_types.py:668-669`: `/project/list` and `/project/info`
   were absent from `LiteLLMRoutes.internal_user_routes`, so even if the
   frontend fired the request, `route_checks.py` returned 403 before the
   handler ran. Fix appends both paths to the list.

3. **Backend querying stale columns**
   `enterprise/litellm_enterprise/proxy/management_endpoints/project_endpoints.py:912-933`:
   `list_projects` was doing
   `find_many(where={"OR": [{"members": {"has": ...}}, {"admins": {"has":
   ...}}]})` against `LiteLLM_TeamTable`, but `team_member_add` no longer
   populates the scalar `members` / `admins` arrays — those are legacy
   columns. The authoritative store is `members_with_roles` (a JSON column
   keyed by role) plus the reverse-index on `LiteLLM_UserTable.teams`. Fix
   switches to one O(1) `find_unique` on the user, reads
   `user_record.teams` for team_ids, then queries projects by those ids.
   `project_info` at `:857-869` gets the same treatment for the 403 guard,
   iterating `team.members_with_roles` and matching `m.user_id ==
   caller_user_id` (with both dict and pydantic-model accessor shapes
   handled by the `isinstance(m, dict)` branch — the right paranoia given
   the JSON column can come back either way depending on prisma version).

## What is load-bearing

- The user-table-first query path at
  `project_endpoints.py:920-928` is a real perf win, not just a
  correctness fix. Prior code did a full scan of `LiteLLM_TeamTable`
  filtered on `members.has` / `admins.has` — both are scalar-array
  containment ops that don't use the team_id index. New code does one
  PK lookup on the user, then one `WHERE team_id IN (...)` on projects,
  which uses the team_id index.
- The `members_with_roles` iteration in `project_info` correctly
  short-circuits on first match (`break`) — important if a future config
  has the same user listed multiple times in `members_with_roles` under
  different roles.
- The frontend `enabled` simplification means the hook now fires for
  *any* logged-in user, including users with zero project memberships.
  The backend correctly returns `[]` for them (the `user_team_ids = []`
  branch yields an empty IN clause → empty result), and the dropdown
  empty-state should already handle that.

## Nits

1. **`internal_user_routes` placement**: the new `/project/list` and
   `/project/info` entries land at the bottom of the `[...]` literal at
   `_types.py:666-669`. The list otherwise groups by surface area (model
   routes together, guardrail routes together, etc.). Consider grouping
   project routes together — purely cosmetic.
2. **Test claim vs. tests added**: PR body says "existing
   `test_project_endpoints_prisma.py` covers changed functions — 16 pass"
   but the diff adds zero new tests. The two changed query patterns
   (`user_record.teams` lookup, `members_with_roles` iteration) are
   distinct from the prior `members`/`admins` paths, so the existing
   tests probably don't cover the new code paths. Recommend adding one
   test per: (a) `list_projects` with a user who has team-memberships
   only via `members_with_roles` (no scalar `members` entry), and (b)
   `project_info` 403 guard for a non-admin team-member (the
   first-match-then-break path).
3. **Race in `project_info`**: `team.members_with_roles or []` handles
   `None` but if the JSON column is malformed (e.g. a string instead of a
   list, or a dict at the top level) the `for m in ...` will iterate
   characters or keys silently. Since this is a 403-guard path, a
   malformed value should fail-closed (reject), not fail-open (skip the
   loop and rely on `is_admin`). Defensive `if not isinstance(team.members_with_roles, list): raise HTTPException(...)` would close that.

## Risk

- Combined-with-frontend behaviour: this PR removes the frontend admin
  gate at the same time as adding the backend route. Without #2 (the
  `internal_user_routes` patch), the frontend would fire and immediately
  401/403, surfacing as a noisier "Failed to load projects" toast for
  every internal user. The two changes are correctly bundled in one PR.
- Backwards compat: any caller relying on the old behaviour where
  `useProjects` silently no-ops for non-admins now gets a real query.
  That's the intended fix.

## Recommendation

Merge after adding the two missing tests (positive `members_with_roles`
path for both `list_projects` and `project_info`). Diagnosis is right,
fix is minimal, no schema migration needed.
