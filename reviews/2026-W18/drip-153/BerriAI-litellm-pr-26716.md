# BerriAI/litellm #26716 — fix(proxy): scope v1 /team/list teams via user.teams

- PR: https://github.com/BerriAI/litellm/pull/26716
- Head SHA: `687dd33d8a3313f840f75b39d348c4f7cf66d0d2`
- Files: `litellm/proxy/management_endpoints/team_endpoints.py`, `tests/test_litellm/proxy/management_endpoints/test_team_endpoints.py`

## Citations

- `team_endpoints.py:4147-4188` — the `_authorize_and_filter_teams` flow is restructured: `is_own_query` computation is hoisted *inside* the `allowed_org_ids is None` branch so it's only evaluated when not an org admin, and `user_id = user_api_key_dict.user_id` defaults are applied when the caller didn't pass a `user_id` (was a 401 path before).
- `team_endpoints.py:4198-4225` — the regular-user (`elif user_id:`) branch is rewritten: the old "fetch all teams via Prisma + filter in Python by `members_with_roles`" approach is replaced with `target_user = await get_user_object(user_id=...)` then `where={"team_id": {"in": user_team_ids}}` scoped query. New 404 raises if the target user isn't found.
- `team_endpoints.py:4228-4231` — the empty-team-list short circuit `if not user_team_ids: return []` saves a needless DB call and avoids `find_many({"team_id": {"in": []}})` which has driver-dependent semantics.
- `tests/.../test_team_endpoints.py:2261+` — new ~220-line test `test_list_team_v1_internal_user_scoped_to_user_table_teams` mocks `get_user_object → LiteLLM_UserTable(teams=["allowed_team"])` and asserts only the canonical-id team comes back (not teams referencing the user only via stale `members_with_roles`).

## Verdict

`merge-after-nits`

## Reasoning

This is a real and worth-fixing scoping bug, and the fix aligns the v1 endpoint with v2's already-canonical behavior, which is the right direction. The original flow had two distinct problems and the patch addresses both:

**Problem 1: the auth gate was wrong.** A non-admin caller hitting `GET /team/list` *without* a `user_id` query param was 401'd because `is_own_query = False` (user_id is None ⇒ comparison fails), even though the natural reading of "list my teams" is exactly the no-arg call. The patch defaults `user_id = user_api_key_dict.user_id` *inside* the gate so the no-arg call is treated as an own-query. This is consistent with how most "list my X" endpoints work, and it correctly preserves the 401 for `?user_id=other_user` calls because then the comparison `user_api_key_dict.user_id == user_id` is False.

**Problem 2: the filtering was over-permissive.** The pre-patch code did `find_many()` (no filter) then post-filtered in Python via `team.members_with_roles`. This exposes teams where the user appears in `members_with_roles` but has been removed from `LiteLLM_UserTable.teams` — i.e. stale denormalization. The fix uses `LiteLLM_UserTable.teams` as the source of truth and scopes the SQL `WHERE team_id IN (...)` accordingly. This matches v2's behavior and removes the asymmetry.

Nits worth surfacing before merge:

1. **Behavior change is silent.** The pre-patch surface was de-facto API: scripts that relied on "list_team returns teams I appear in via members_with_roles even after my user row was scrubbed" will now see fewer rows. This isn't documented in the PR body. A migration note in `litellm-proxy-extras` or the changelog is worth adding — not blocking, but courteous.

2. **The 404 is debatable.** When `target_user is None` (or `get_user_object` raises `ValueError`), the patch returns 404 with `f"User not found, passed user_id={user_id}"`. For an internal user whose own session somehow points at a deleted user row, surfacing 404 instead of `[]` is arguably right (it surfaces a real stale-session bug) but it's also a behavior change for callers that previously got `[]`. Consider gating the 404 on "user_id was explicitly passed and didn't match `user_api_key_dict.user_id`" so an own-query against a torn user row degrades gracefully.

3. **Test naming.** `test_list_team_v1_internal_user_scoped_to_user_table_teams` is good; consider also a negative test asserting that a team listed only in stale `members_with_roles` (and not in `LiteLLM_UserTable.teams`) is *excluded*. The current test asserts inclusion of the canonical row, but the regression of interest is exclusion of the stale row — those are not the same assertion.

4. The `try/except ValueError → 404` then immediate `if target_user is None → 404` (lines 4214-4220) duplicates the same response shape twice; collapse into a single helper or one combined branch.

Direction is right; merge after the migration note + the negative test.
