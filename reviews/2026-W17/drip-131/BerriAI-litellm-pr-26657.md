# Review — BerriAI/litellm #26657

- **Title**: fix(projects): allow internal users to view projects
- **Author**: ishaan-berri
- **Head**: `57d6500d5fc286b7f435c7c98a518e242d84d18e`
- **Verdict**: needs-discussion

## Summary

Three layered fixes to make `internal_user`-role users see their projects: (1) `project_endpoints.py` switches the membership lookup for `/project/list` and `/project/info` from querying `LiteLLM_TeamTable.members`/`.admins` to reading `LiteLLM_UserTable.teams` (the column actually populated by `team_member_add`); (2) `_types.py` adds `/project/list` and `/project/info` to the `internal_user_routes` allowlist so requests aren't rejected before reaching the endpoint; (3) `useProjects.ts` extends the React Query `enabled` gate to fire for `internalUserRoles` in addition to `all_admin_roles`.

## Findings

- **Same root-cause as drip-129/#26659** (`fix(projects): non-admin users can now see their projects`, head `7be66dda`). That PR also identifies the schema-vs-code mismatch where `team_member_add` writes to `LiteLLM_UserTable.teams` while `/project/list`+`/project/info` were querying `LiteLLM_TeamTable.members`/`.admins`. Both PRs are open, both touch the same three files (`project_endpoints.py`, `_types.py`, `useProjects.ts`), and both flip the same query and the same allowlist. **One must be closed in favor of the other** before either lands; otherwise the second to merge will produce a no-op or merge conflict.
- **Allowlist diff differs subtly from #26659**: this PR (`_types.py:665-668`) appends to `LiteLLMRoutes` membership inline, while #26659 added the same routes to `self_managed_routes`. The user-facing effect *should* be the same (both grant non-admin access) but the routes-classification semantics differ — `self_managed_routes` typically means "routes a user can call to manage their own data" while the bare `internal_user_routes` membership shown in the diff context (`spend_tracking_routes + key_management_routes` neighborhood) means "routes available to internal_user role". Maintainer should pick the right classification bucket; #26659's `self_managed_routes` placement matches the read-your-own-projects intent better.
- **`is_team_member` collapse at `:855-862`** correctly reads `user_row.teams` and checks `project.team_id in user_row.teams` instead of doing two separate `in` checks against `team.admins` and `team.members`. Functionally identical for the membership question but loses the admin-vs-member distinction — if any downstream code branches on `is_team_admin`, this collapse drops the signal silently. Search for `is_team_admin`/`team.admins` references before merging.
- **`list_projects` rewrite at `:911-920`** correctly handles the `not user_api_key_dict.user_id` empty path by initializing `team_ids: List[str] = []` and conditionally populating; the `find_many({"team_id": {"in": []}})` downstream call returns empty rather than crashing. Good defensive shape.
- **No tests in the diff** — neither a regression test for the schema-vs-code mismatch (the strongest possible protection: `mock_prisma.db.litellm_teamtable.find_many.assert_not_called()` like #26659 has) nor a behavioral test for the allowlist change. PR's own pre-submission checklist names "Adding at least 1 test is a hard requirement" and the box is unchecked.
- **Frontend `enabled` gate at `useProjects.ts:82-86`** correctly disjuncts admin-roles and internal-user-roles; the formatting expansion (one-line → multi-line) is fine. Single nit: the implicit-empty-string-on-undefined `(userRole || "")` is repeated three times — could collapse to a `const role = userRole || ""` local but cosmetic.
- **PR description's Pre-Submission checklist is entirely unchecked** including the Greptile-review-with-4/5-confidence requirement, suggesting this is a draft-quality submission rather than a ready-to-merge.

## Recommendation

Hold pending coordination with #26659 — the two PRs are duplicate fixes for the same root cause with materially different allowlist-classification choices. Maintainer should pick one PR (recommend #26659 since it has the regression test asserting `teamtable.find_many.assert_not_called()` and the `self_managed_routes` placement matches the intent), close the other, and require the surviving PR to add tests before merging. The underlying technical analysis in both PRs is correct; the duplication is the blocker.