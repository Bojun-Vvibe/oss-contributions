# BerriAI/litellm #26988 — fix(auth): prevent privilege escalation via common_checks DB fallback

- URL: https://github.com/BerriAI/litellm/pull/26988
- Head SHA: `33bc150c50a95502ba01eef194490aeb98a82151`
- Author: @qi-scape
- Stats: +60 / -13 across 2 files

## Summary

Closes a privilege-escalation hole in `user_api_key_auth`: previously,
if `_run_centralized_common_checks` raised **any** exception (including
a transient DB connectivity error), the catch-all branch routed through
`UserAPIKeyAuthExceptionHandler._handle_authentication_error`, which
under DB-down conditions can mint an unrestricted fallback token —
silently stripping the budget/model/team restrictions that the
**already-authenticated** caller's token carried. The fix splits the
handler into three branches: authorization denials propagate as
`ProxyException`, DB-connectivity errors fall through with the original
token preserved, and other unexpected errors keep the legacy path.
Also fixes a missing `await` on a cache lookup.

## Specific feedback

- `litellm/proxy/auth/auth_checks.py:2521` — `cached_org_obj =
  await user_api_key_cache.async_get_cache(key=cache_key)`. This was a
  latent bug: without `await`, `cached_org_obj` was a coroutine, the
  `is not None` check always passed, and the subsequent `isinstance(...,
  dict)` branch silently fell through. Worth a separate test that
  specifically verifies a cache-hit path returns the cached
  `LiteLLM_OrganizationTable`.
- `litellm/proxy/auth/user_api_key_auth.py:1949-1961` — the new
  `(HTTPException, ProxyException, litellm.BudgetExceededError)`
  branch correctly wraps `BudgetExceededError` in a `ProxyException`
  with `code=400`. Sanity check the exception type ordering: Python
  evaluates left-to-right but `isinstance` does not short-circuit on
  the tuple-match itself — only the subsequent `if isinstance(e,
  litellm.BudgetExceededError)` does. Current ordering is fine.
- `litellm/proxy/auth/user_api_key_auth.py:1962-1991` — the
  `PrismaDBExceptionHandler.is_database_connection_error(e)` gate is
  the heart of the fix. Two concerns:
  1. The classifier itself (`PrismaDBExceptionHandler.is_database_connection_error`)
     is doing a lot of load-bearing work here. If it ever returns
     `False` for a real DB outage (e.g. a new `prisma`-client error
     class), the request falls through to the legacy
     `_handle_authentication_error` path and the privilege-escalation
     window reopens. Consider failing **closed** for any unrecognised
     exception during authorization checks (i.e. raise 500), rather
     than handing back to the legacy fallback handler.
  2. The fall-through silently proceeds with `user_api_key_auth_obj` —
     please add a regression test that spawns an authenticated request
     with a budgeted/model-restricted token, simulates a DB drop in
     `_run_centralized_common_checks`, and asserts the request still
     respects the original `models` allowlist downstream.
- `litellm/proxy/auth/user_api_key_auth.py:1980` — `verbose_proxy_logger.warning`
  with raw `str(e)`. If the prisma error message ever contains the DB
  URI (some drivers do), this would leak credentials to logs. Consider
  wrapping with the existing log redactor.
- The change wisely keeps the `verbose_proxy_logger.warning` at WARN
  rather than ERROR — DB blips are operationally noisy without being
  truly errored when the auth surface degrades gracefully.
- Docstring/comment block at `:1923-1939` is excellent — explains
  exactly what the threat model is. Keep this.
- No test changes in the diff. For a security-relevant fix this is the
  primary gap. At minimum: one test per branch (auth-denial → 4xx,
  DB-down → original token preserved, unexpected exception → 5xx).

## Verdict

`merge-after-nits` — fix is correct and the threat model is articulated
clearly. **Strongly recommend** adding the regression test for the
"DB drop preserves original restrictions" path before merge — this is
exactly the kind of fix that a future refactor will silently undo.
Consider failing closed on unrecognised exception types as defence in
depth.
