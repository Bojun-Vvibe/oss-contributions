# BerriAI/litellm#26649 — fix(projects): internal users can see projects in Create Key dialog

- **Repo**: BerriAI/litellm
- **PR**: [#26649](https://github.com/BerriAI/litellm/pull/26649)
- **Head SHA**: `55298f595824521a7cf43365d1c69c6041c8bb4e`
- **Reviewed**: 2026-04-29
- **Verdict**: `merge-as-is`

## Context

Three-layer bug where internal-role users (and team members) could
never see their team's projects in the dashboard's Create Key dialog,
even when their team had associated projects. PR identifies all three
layers and fixes each one — exactly the kind of "I traced this all
the way down" diff worth approving on first read.

## The three bugs (with fixes)

### Bug 1 — UI: useQuery gated on admin role

`ui/litellm-dashboard/src/app/(dashboard)/hooks/projects/useProjects.ts`

Old:
```ts
enabled: Boolean(accessToken) && all_admin_roles.includes(userRole || "")
```

`internal_user` isn't in `all_admin_roles`, so the `/project/list`
fetch was never even fired for those callers — the spinner just
never resolved.

Fix:
```ts
enabled: Boolean(accessToken)
```

Drops the role gate entirely. Trust falls to the server-side route
allow-list (Bug 2) instead of duplicating it in the UI. Right
direction — UI-side role gates are notoriously divergent from
server-side authz and tend to lag.

### Bug 2 — `/project/list` and `/project/info` not in `internal_user_routes`

`litellm/proxy/_types.py`

```python
LiteLLMRoutes.internal_user_routes adds:
"/project/list",
"/project/info",
```

Without these, an internal_user calling either endpoint got 401
*before* the handler ran. Even with Bug 1 fixed, Bug 2 alone would
have kept the dialog empty. Both have to land together.

### Bug 3 — Membership lookup queried wrong column

`enterprise/litellm_enterprise/proxy/management_endpoints/project_endpoints.py`

The bug:
```python
team = await prisma_client.db.litellm_teamtable.find_unique(...)
is_team_member = (
    user_api_key_dict.user_id in team.admins
    or user_api_key_dict.user_id in team.members
)
```

`LiteLLM_TeamTable.members` and `.admins` are *never populated* —
`team_member_add` writes to `LiteLLM_UserTable.teams` instead. So the
membership check always returned False for non-admin users, even when
they were genuinely on the team.

The fix:
```python
user_row = await prisma_client.db.litellm_usertable.find_unique(
    where={"user_id": user_api_key_dict.user_id}
)
user_teams = user_row.teams if (user_row and user_row.teams) else []
is_team_member = project.team_id in user_teams
```

`list_projects` gets the parallel fix:

```python
team_ids: list[str] = []
if user_api_key_dict.user_id:
    user_row = await prisma_client.db.litellm_usertable.find_unique(
        where={"user_id": user_api_key_dict.user_id}
    )
    team_ids = user_row.teams if (user_row and user_row.teams) else []

projects = await prisma_client.db.litellm_projecttable.find_many(
    where={"team_id": {"in": team_ids}},
    ...
)
```

Same shape both places. Comment in both spots explicitly says
"membership is written to LiteLLM_UserTable.teams by team_member_add,
not to LiteLLM_TeamTable.members which is always empty" — exactly
the kind of inline note that prevents this from being re-introduced
during a future "regularize the queries" refactor.

## Test coverage

Two new tests in
`tests/enterprise/litellm_enterprise/proxy/management_endpoints/test_project_endpoints_prisma.py`:

1. `test_project_routes_in_internal_user_routes` — direct enum
   inspection: asserts `/project/list` and `/project/info` are in
   `LiteLLMRoutes.internal_user_routes.value`. Cheap regression
   guard against someone accidentally removing them.

2. `test_list_projects_internal_user_queries_user_table_teams` —
   patches `prisma_client`, calls `list_projects` with an
   `internal_user` auth, asserts:
   - `find_unique` was called on `litellm_usertable` with the
     correct `user_id`
   - `find_many` on `litellm_projecttable` was called with
     `where={"team_id": {"in": [team_id]}}` — i.e. the team list
     pulled from the user table actually flowed into the project
     query

   This is exactly the right test for Bug 3 — it locks in *which
   table* gets queried, not just "the function returned non-empty".
   A future refactor that switches back to
   `litellm_teamtable.members` would fail this test even if the
   end-to-end behavior happened to coincidentally still work in
   some test fixture.

## Risk analysis

- **Backwards-compat**: admins were already getting the right data
  via the `is_admin` early-return path; their flow is unchanged.
  Internal users now see projects; previously they saw nothing.
  No path that worked before now breaks.
- **Performance**: one extra `find_unique(LiteLLM_UserTable)` per
  list-projects call for non-admin callers. Single-row indexed
  lookup, negligible. Same as the membership check pattern used
  elsewhere in the codebase.
- **Authz boundary**: now that the UI fetches `/project/list` for
  every authenticated user, the server *must* authoritatively
  filter to the user's own teams. Bug 3's fix does that
  correctly — the query is `team_id in user_row.teams`. A user
  with no teams gets `team_ids = []` → an `IN ()` query → empty
  result. Safe.
- **Empty-teams edge case**: `user_row.teams if (user_row and
  user_row.teams) else []` correctly handles three cases: user not
  in DB (shouldn't happen post-auth, but defensive), user with
  `teams = None`, user with `teams = []`. All collapse to "no
  projects visible".

## Suggestions (non-blocking)

- The "members column is always empty" property is now relied on
  by *not* querying it. Worth an assertion or schema-level note
  somewhere that prevents someone from "fixing" the schema by
  populating the column without understanding that the UserTable
  side is the source of truth. Cleanup PR territory — not blocking.

## Verdict

`merge-as-is`. Three independent bugs in three layers, each
correctly identified with the right narrow fix, with test coverage
that locks in the *which-query* answer for the most subtle of the
three. The inline comments about `LiteLLM_TeamTable.members` being
empty are gold — exactly the institutional knowledge that needs to
live in code, not in a Slack message.

## What I learned

"User says X is broken" → bug-finder finds *one* layer that's
broken → fixes that one → user reports a different symptom because
layer 2 is still broken → repeat. The discipline of tracing all
the way down before opening a PR (UI fetch gate → route allow-list
→ underlying query correctness) is what turns a one-bug PR into a
one-PR fix. Worth modeling: when you find layer-1 broken, *keep
going* until you've replicated the actual failure end-to-end.
