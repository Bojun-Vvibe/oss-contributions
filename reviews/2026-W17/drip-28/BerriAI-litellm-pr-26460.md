# PR #26460 — feat(proxy): Add cleanup job for expired LiteLLM dashboard session keys

- Repo: BerriAI/litellm
- Head SHA: 69c5840e5fe821e202e5efbee86fb274f4fc4b4d
- Files touched: 4 (+437 / -2)

## Summary

Adds an env-flagged APScheduler background job that periodically
deletes expired `team_id == "litellm-dashboard"` virtual keys
(UI session tokens). Three new env vars
(`LITELLM_EXPIRED_UI_SESSION_KEY_CLEANUP_ENABLED`,
`..._INTERVAL_SECONDS` default 86400, `..._BATCH_SIZE` default
1000), one new manager class
(`ExpiredUISessionKeyCleanupManager` in
`litellm/proxy/common_utils/expired_ui_session_key_cleanup_manager.py`),
scheduler wiring in
`litellm/proxy/proxy_server.py:_initialize_expired_ui_session_key_cleanup_background_job`,
and 5 mocked unit tests in
`tests/test_litellm/proxy/common_utils/test_expired_ui_session_key_cleanup_manager.py`.

## Specific findings

- `litellm/constants.py:1396-1404` adds the three env-flag
  constants. `..._ENABLED` reads as a string `"false"` default;
  the proxy_server wiring does the truthy check (not visible in
  the diff window — reviewer should confirm it matches the
  pattern other LITELLM_*_ENABLED flags use, e.g.
  `str(...).lower() == "true"` rather than Python `bool(str)`
  which is always `True` for non-empty strings).
- `expired_ui_session_key_cleanup_manager.py:31-79` — the
  cleanup path:
  1. Pulls a `PodLockManager.acquire_lock(cronjob_id=
     EXPIRED_UI_SESSION_KEY_CLEANUP_JOB_NAME)` if a Redis-backed
     lock manager is provided. Bails out (returns 0) if the lock
     can't be acquired.
  2. `_find_expired_ui_session_keys` queries
     `litellm_verificationtoken.find_many(where={"team_id":
     UI_SESSION_TOKEN_TEAM_ID, "expires": {"lt": now}},
     take=BATCH_SIZE)` — uses `datetime.now(timezone.utc)`,
     which is the right thing.
  3. Deletes via the existing `delete_verification_tokens(...)`
     helper, then fires
     `KeyManagementEventHooks.async_key_deleted_hook(...)` for
     audit logging. Reusing both is correct — preserves cache
     invalidation and audit trail.
  4. `system_user =
     UserAPIKeyAuth.get_litellm_internal_jobs_user_api_key_auth()`
     attributes the deletes to `litellm_internal_jobs`. PR body
     confirms `deleted_by = system` and
     `litellm_changed_by = litellm_internal_jobs` show up in the
     deleted-token row.
- `expired_ui_session_key_cleanup_manager.py:55-58` —
  `verbose_proxy_logger.info("Deleted %s expired UI session
  key(s)", len(tokens))` uses lazy formatting (good — no
  string-format work when the level is filtered).
- `expired_ui_session_key_cleanup_manager.py:81-87` — the
  finally-block releases the Redis lock only if it was acquired.
  Standard pattern, correct.
- `proxy_server.py:6740-6743` adds three `global` declarations
  (`prisma_client`, `proxy_logging_obj`, `user_api_key_cache`)
  to `_initialize_spend_tracking_background_jobs`, which is
  fine — they were already declared in the inner branch — but
  hoisting them silently changes the scoping for any future
  edits to that function. A short comment would prevent
  accidental shadowing later.

## Risks / nits

- **Single-batch-per-tick**: `_find_expired_ui_session_keys`
  calls `find_many(... take=BATCH_SIZE)` and the tick runs
  every `INTERVAL_SECONDS` (default 24h). If the steady-state
  expired-key rate exceeds 1000/day, the cleanup will fall
  behind permanently. Worth either looping until empty within
  the same tick (cheap on a Redis lock) or documenting the
  rate ceiling explicitly.
- **`team_id` string is shared with auth code paths**:
  `UI_SESSION_TOKEN_TEAM_ID = "litellm-dashboard"` is the only
  thing distinguishing dashboard session keys from real virtual
  keys with `team_id="litellm-dashboard"`. If an operator ever
  creates a real team with that exact id, this job will start
  deleting that team's keys. Worth a defensive check on
  `key_alias` or another sentinel field (or a doc warning that
  `litellm-dashboard` is reserved).
- **Audit log fires per batch, not per token**: the
  `KeyManagementEventHooks.async_key_deleted_hook(data=
  KeyRequest(keys=tokens), ...)` call passes the whole token
  list. If 1000 keys are deleted, that's a single audit event
  rather than 1000 — fine for volume but downstream consumers
  that key audit events to single tokens will lose granularity.
  PR body screenshot suggests it's a single combined event.
- Tests are in the right location and look comprehensive
  (5 cases — query filter, deletion, no-op when empty, lock
  contention paths). Reviewer should confirm the lock-not-
  acquired path test asserts the early `return 0` and *doesn't*
  call `delete_verification_tokens`.

## Verdict

**merge-after-nits** — Solid manager class with correct
lock + audit + cache-invalidation reuse. Address the
single-batch-per-tick rate ceiling (loop or document), add
the `litellm-dashboard` reserved-team-id warning, and confirm
the env-flag truthy check uses the project's standard pattern.

## What I learned

A periodic cleanup job that targets rows by a magic
`team_id` string is one rename or one operator misconfig away
from deleting real tenant data. The defense in depth here is
to either reserve the magic id at the schema level (unique
constraint that prevents real teams from claiming it) or
require a *second* sentinel (e.g. `key_alias LIKE
'session-%%'`) so a single bad config can't expose the
delete primitive.
