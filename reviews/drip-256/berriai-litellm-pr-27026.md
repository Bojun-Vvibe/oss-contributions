# BerriAI/litellm PR #27026 — [Fix] Honor team_member_permissions /key/list in /key/list endpoint

- URL: https://github.com/BerriAI/litellm/pull/27026
- Head SHA: `b8cf48a10237e150c908b9394d3fce2294a42899`
- Author: yuneng-jiang
- Verdict: **merge-after-nits**

## Summary

Bug-fix in `litellm/proxy/management_endpoints/key_management_endpoints.py`: previously, granting a non-admin team member the `/key/list` permission via `team_member_permissions` did not actually expand their visibility — they still only saw their own keys plus service-account keys. This PR introduces `_get_team_ids_with_key_list_permission_from_objects(...)` and merges those team IDs into `admin_team_ids` so the existing helper treats them as full-visibility teams.

## Line-level observations

- `key_management_endpoints.py` line 60 adds `_team_member_has_permission` to the existing import block from the same module — clean, no new module dependency.
- New helper at lines ~4439–4458 filters `team_objects` to non-admin teams where `_team_member_has_permission(..., permission=KeyManagementRoutes.KEY_LIST.value)` returns true. Correct: it intentionally excludes admins (who already get full visibility via `_get_admin_team_ids_from_objects`) to avoid double-counting.
- Lines ~4610–4623 (inside `list_keys`): the new `list_permission_team_ids` is computed and `admin_team_ids = list({*admin_team_ids, *list_permission_team_ids})`. Using set-union dedupes correctly; the resulting list ordering is non-deterministic, which is fine for downstream `IN (...)` queries but worth noting if any caller relies on order.
- The new tests in `tests/test_litellm/proxy/management_endpoints/test_key_management_endpoints.py` (lines ~5024–5300+) are well-structured: `_invoke_list_keys_and_capture_helper_kwargs` patches `_list_key_helper` to capture `admin_team_ids` / `member_team_ids`, then asserts classification rather than full end-to-end output. This is exactly the right test seam — it pins the contract of the classification step without coupling to the SQL helper internals.
- The positive test `test_list_keys_team_member_with_key_list_permission_sees_all_team_keys` and the negative test `test_list_keys_team_member_without_key_list_permission_only_service_accounts` give round-trip coverage of the regression.

## Concerns

- The fix re-uses `admin_team_ids` as the carrier for "full key visibility." Semantically the user is **not** an admin — if any downstream code branches on `admin_team_ids` to *grant write/delete/regenerate* (not just list), that user just got escalated. A grep for other consumers of `admin_team_ids` in the helper chain is warranted before merge. Safer long-term: introduce a third bucket (`full_visibility_team_ids`) and OR it in only at the SQL filter for `list`.
- Edge case: if `team_member_permissions` is set on the team but the caller is *not* a member of it (`_fetch_user_team_objects` shouldn't return such teams, but defense-in-depth matters), the helper would still tag the team. Consider asserting membership inside the new helper.

## Suggestions

1. Confirm `admin_team_ids` isn't consumed by any non-list code path before piggybacking on it; otherwise add a separate `list_visibility_team_ids` carrier.
2. Add an explicit membership check inside `_get_team_ids_with_key_list_permission_from_objects` so a stale team object can't grant visibility.
3. Add one more test: a team where the user is in `members_with_roles` but is **also** a team admin — assert no double-classification (the team appears in `admin_team_ids` exactly once).
