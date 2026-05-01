# PR #26988 — fix(auth): prevent privilege escalation via common_checks DB fallback

- Repo: BerriAI/litellm
- Author: qi-scape
- Head: `a2391a6be90e6cd9b101255e36f323e1edf0b505`
- URL: https://github.com/BerriAI/litellm/pull/26988
- Verdict: **merge-after-nits**

## What lands

Two-file +47/-13 closing a privilege-escalation hole in
`litellm/proxy/auth/user_api_key_auth.py`. Pre-PR, when
`_run_centralized_common_checks()` raised any exception (including a
transient DB error), the code unconditionally routed it through
`UserAPIKeyAuthExceptionHandler._handle_authentication_error()`, which on
the DB-unavailable path issues an unrestricted fallback token —
*replacing* the already-validated token whose budget/model-access/team
restrictions the request was supposed to be honoring. PR splits the
exception handling into three branches.

Plus a one-line drive-by fix in `litellm/proxy/auth/auth_checks.py:2521`
adding the missing `await` to `user_api_key_cache.async_get_cache(...)`.

## Specific findings

- `auth_checks.py:2521` — adding `await` to `async_get_cache()` is a real
  bug: without `await`, `cached_org_obj` was a coroutine object, never
  None, and the `if cached_org_obj is not None` check at `:2522` always
  passed, then `isinstance(cached_org_obj, dict)` at `:2523` returned
  False, then the function would attempt `LiteLLM_OrganizationTable(**...)`
  on a coroutine — likely raising. Worth confirming this path was actually
  unreachable before (e.g. cache miss is the common case so the bug masked
  itself), otherwise this single-line fix deserves its own PR with a
  regression test. As-is the fix is correct but bundled.
- `user_api_key_auth.py:1923-1939` is the new docstring block. It is
  actually load-bearing: explains the threat model (transient DB error
  during *secondary* checks must not erase the *primary* token's
  restrictions). This is exactly the kind of comment that prevents the
  next refactor from re-introducing the bug. Keep it.
- The new exception handling at `:1946-1948`:
  ```python
  except (HTTPException, ProxyException, litellm.BudgetExceededError):
      # Authorization denial — propagate as-is so the caller gets 4xx.
      raise
  ```
  is the right shape — denial signals propagate, never get rewritten as
  fallback tokens. The set `{HTTPException, ProxyException,
  BudgetExceededError}` covers the canonical denial types but worth
  asking: are there other denial types raised from common_checks? E.g.
  `litellm.exceptions.AuthenticationError`,
  `litellm.exceptions.RateLimitError`? If so, those would currently fall
  into the `except Exception` branch and (if they're not classified as
  DB-connection errors) get routed to the fallback handler — which is
  still better than pre-PR but not as defensive as it could be.
- `:1965-1971` — the actual fix, where DB-connection errors are *logged*
  via `verbose_proxy_logger.warning()` and then control falls through to
  return `user_api_key_auth_obj` with original restrictions intact. Two
  things to nail down:
  - The fall-through is implicit (no explicit `pass` or comment in the
    final placement). Subsequent maintainers refactoring this block may
    not realize the silence here is the whole point. A `# fall through:
    keep original token` comment at the `if` block exit would help.
  - `verbose_proxy_logger.warning(...)` is the right level for "request
    succeeded but with degraded enforcement"; consider also incrementing
    a metric (`litellm_auth_db_fallback_total` or similar) so this
    condition shows up on dashboards rather than only in noisy logs.
- `:1976-1984` non-DB error path still routes through
  `_handle_authentication_error`. Correct — coding bugs in common_checks
  should still surface as 4xx via the existing handler, not silently
  fall through.
- The local import `from litellm.proxy.db.exception_handler import
  PrismaDBExceptionHandler` at `:1965` inside the `except` block is fine
  for avoiding a top-level import cycle but worth confirming it doesn't
  exist at module top already (cheap).

## Risks

- Behavior change for *deployments that relied on the unrestricted
  fallback being issued whenever common_checks couldn't reach the DB*.
  These deployments would now get the restricted token back — which is
  the secure behavior, but may surface as "users suddenly hitting budget
  limits during DB outages." Worth a release-note callout: "security fix
  may tighten enforcement during DB outages."
- No tests in this diff. For a privilege-escalation fix, at minimum need:
  (1) common_checks raises a DB-connection error → original token
  preserved with its restrictions; (2) common_checks raises ProxyException
  → propagated as 4xx; (3) common_checks raises a coding-bug exception
  → routed through the existing handler.

## Verdict

**merge-after-nits**. Fix is correct and the docstring at `:1923-1939`
makes the threat model explicit, but ship blockers are: add the three
regression tests above, split the `auth_checks.py` `await` fix into its
own PR (or document why it's bundled), and document the deployment
behavior change in the release notes.
