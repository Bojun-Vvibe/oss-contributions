# BerriAI/litellm#26466 — Team-level guardrails auto-apply alongside global policy guardrails

- PR: https://github.com/BerriAI/litellm/pull/26466
- Author: shivamrawat1 (Shivam Rawat)
- +261 / -2
- Head SHA: `24e24d4e302c6bf799f262df05a40af0724c4091`

## Summary

Fixes two interacting bugs that caused team-direct guardrails to be silently
skipped when a global policy guardrail (scope=`*`) was also configured:

1. **Auth caching bug.** After "Check 6" in `_user_api_key_auth_builder`
   freshly fetched a team object from DB/cache, only `team_object_permission`
   was copied onto `valid_token`; `team_metadata` was never refreshed, so any
   guardrail added to the team after the API key was first cached stayed
   invisible until the cache evicted.
2. **Guardrail metadata-resolution bug.** `get_guardrail_from_metadata` read
   `data["litellm_metadata"]` before `data["metadata"]`, so a non-empty
   `litellm_metadata` without a `guardrails` key shadowed the merged list that
   `move_guardrails_to_metadata` had correctly written to `data["metadata"]`.

## Specific findings

- `litellm/proxy/auth/user_api_key_auth.py:1462` (SHA
  `24e24d4e302c6bf799f262df05a40af0724c4091`) — the new
  `valid_token.team_metadata = _team_obj.metadata` assignment is the load-
  bearing fix. Placed inside the existing `if _team_obj is not None:` block
  next to the `team_object_permission` assignment, so it benefits from the
  same null-guard. Worth confirming `_team_obj.metadata` is always a `dict`
  (not `None`) on the cached path — the test mocks a `dict` directly so this
  isn't exercised.
- `litellm/integrations/custom_guardrail.py:333` — the rewritten loop
  iterates `("metadata", "litellm_metadata")` in order and returns the first
  dict that actually contains a `guardrails` key. The `isinstance(meta, dict)`
  guard is defensive against callers stuffing `None` or non-dict values into
  either field. The order is important and matches the comment: regular
  endpoints land in `metadata`, thread/assistant endpoints in
  `litellm_metadata`.
- `tests/test_litellm/proxy/test_litellm_pre_call_utils.py:3372` — the
  regression test pre-populates `data["litellm_metadata"] = {"some_user_field":
  "some_value"}` *without* a `guardrails` key, which is the exact shape that
  used to shadow the merged list. Good coverage of the original failure mode.
- `tests/test_litellm/proxy/auth/test_user_api_key_auth.py:1735` — the
  `test_team_metadata_refreshed_from_team_object_during_auth` test asserts
  `result.team_metadata == {"guardrails": ["test-guardrail-333"]}` after the
  auth flow runs, locking in the "fresh team object overwrites stale cached
  team_metadata" contract. This is the right level to assert at.

## Verdict

`merge-after-nits`

## Rationale

Both fixes are correct and minimal, and the regression coverage is good. One
small concern at `user_api_key_auth.py:1462`: blindly assigning
`_team_obj.metadata` onto `valid_token.team_metadata` overwrites any caller-
side mutations that previously survived the auth flow on a cache hit. If
anything downstream of "Check 6" mutates `valid_token.team_metadata` and
relies on it surviving across requests (it shouldn't — that would be a
different bug), this change would silently drop those mutations. A quick
grep for `valid_token.team_metadata =` callsites would settle it. Otherwise,
ship.
