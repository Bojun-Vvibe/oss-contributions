# Review — BerriAI/litellm#26994

- PR: https://github.com/BerriAI/litellm/pull/26994
- Title: feat(scim): add opt-in `auto_create_keys_for_scim_users` flag
- Head SHA: `e3325193d806110602f6a467940bc178887fd71d`
- Size: +209 / −13 across 2 files
- Verdict: **merge-after-nits**

## Summary

Adds a new boolean `litellm_settings.auto_create_keys_for_scim_users`
that, when explicitly set to `true`, causes both SCIM `POST /Users`
and group-membership-driven user creation to also auto-generate an
API key for the new user. Defaults to `False` so existing SCIM
deployments are unaffected.

## Evidence / specific spots

- `litellm/proxy/management_endpoints/scim/scim_v2.py`:
  - The existing `_get_scim_upsert_user_setting()` is refactored into a
    generic `_get_litellm_setting(setting_name, default)` helper
    (lines 209-231); the old setter delegates to it for backward compat.
  - New `_get_auto_create_key_for_scim_user_setting()` (lines 245-256)
    returns the new flag, defaulting to `False`.
  - In `_create_user_if_not_exists` (line 391): `auto_create_key =
    await _get_auto_create_key_for_scim_user_setting()` then
    `NewUserRequest(... auto_create_key=auto_create_key ...)`.
  - Same change in the `create_user` endpoint (line 900).
- New tests `test_create_user_auto_create_key_false_by_default` etc.
  in `tests/test_litellm/proxy/management_endpoints/scim/test_scim_v2_endpoints.py`
  exercise both default and opt-in paths.

## Notes / nits

- The default-false choice is correct from a security posture
  standpoint; the README/docs change for the new setting is not in this
  PR — make sure the flag is documented in the SCIM admin docs before
  release notes go out, otherwise users will discover it via grep.
- `_get_litellm_setting` swallows all exceptions and returns `default`.
  Fine, but if the proxy config is genuinely unreadable, every
  per-request lookup will silently log a warning and hide a real
  outage. Worth memoizing the result for the lifetime of a process
  (or at least debouncing the warning) to avoid log spam.
- Both call sites do `await _get_auto_create_key_for_scim_user_setting()`
  inside the request handler — that's two awaits per SCIM POST. Same
  memoization point applies for performance.
- API-key generation has security implications (where is the key
  returned? who can read it?). Reviewer should confirm the SCIM
  response shape doesn't accidentally surface the raw key to whoever
  initiated the SCIM request — if it does, the docs need a clear
  callout.

Functionally clean. Merge after docs + a brief look at whether the
generated key is exposed anywhere unintentionally.
