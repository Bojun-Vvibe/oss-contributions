# BerriAI/litellm #26929 — fix(proxy): reject user_id=None on non-admin analytics endpoints (cross-tenant disclosure)

- **Repo:** BerriAI/litellm
- **PR:** https://github.com/BerriAI/litellm/pull/26929
- **HEAD SHA:** `0f98f3754fd960429b5b7ad937cb88c16fb2b315`
- **Author:** michelligabriele
- **Verdict:** `merge-as-is`

## What the diff does

Multi-file CVE-shaped cross-tenant disclosure fix on three analytics
endpoints (`POST /usage/ai/chat`, `GET /user/daily/activity`,
`GET /user/daily/activity/aggregated`):

1. **New shared guard helper.**
   `litellm/proxy/management_endpoints/common_utils.py:34-58` adds
   `require_caller_user_id_for_non_admin(user_api_key_dict) -> str` which
   returns the caller's `user_id` or raises
   `HTTPException(status_code=403)` with a typed `detail` object naming
   the failure ("Service-account keys cannot query user analytics. Use a
   user-bound key, or call as a proxy admin."). Docstring explicitly names
   the load-bearing precondition: *"Callers must check is_admin first;
   this helper is only valid on the non-admin scoping branch."*

2. **`/usage/ai/chat`** at
   `litellm/proxy/management_endpoints/usage_endpoints/endpoints.py:53-57`
   — splits the prior unconditional `user_id = user_api_key_dict.user_id`
   into `if is_admin: user_id = user_api_key_dict.user_id` else
   `user_id = require_caller_user_id_for_non_admin(user_api_key_dict)`.

3. **`/user/daily/activity` + aggregated** at
   `internal_user_endpoints.py:2588-2593` and `:2686-2691` — replaces the
   silent fallback `if user_id is None: user_id = user_api_key_dict.user_id`
   followed by `if user_id != user_api_key_dict.user_id: raise 403` (the
   same-user check) with `caller_user_id =
   require_caller_user_id_for_non_admin(user_api_key_dict)` then `if
   user_id is None: user_id = caller_user_id` and `if user_id !=
   caller_user_id: raise 403`. The same-user check now compares against a
   guaranteed-non-`None` value.

4. **Defense-in-depth tripwire.**
   `litellm/proxy/management_endpoints/usage_endpoints/ai_usage_chat.py:443-451`
   adds `if user_id is None: raise ValueError("Non-admin caller has
   user_id=None; refusing to issue an unscoped query. Endpoint-level
   guard missing.")` on the non-admin branch of `_resolve_fetch_kwargs`.

5. **Tests** — three test files:
   - `test_common_utils.py:486-519` — `TestRequireCallerUserIdForNonAdmin`
     pins the helper: returns `user_id` when present, raises 403 when
     `None`, and the assertion checks both `status_code == 403` and that
     `"Service-account keys"` is in the detail (so the error-message
     contract can't drift silently).
   - `test_internal_user_endpoints.py:1735-1834` — two new tests
     `test_get_user_daily_activity_rejects_service_account_caller` and the
     aggregated twin, both with `mock_get_daily.assert_not_called()` as the
     load-bearing tripwire-pin so the entry guard's "never reach the
     daily-activity builder" contract is enforced end-to-end.
   - `test_ai_usage_chat.py:413-468` — `TestUsageAiChatServiceAccountGuard`
     covering the endpoint-level reject and the defense-in-depth
     `_resolve_fetch_kwargs` tripwire.

## Why the change is right

The bug analysis in the PR body is the textbook "fail-closed at the
intent layer, not the value layer" lesson:

- The daily-activity SQL builder treats `entity_id is None` as **"skip the
  entity filter entirely"** — appropriate for admin global views,
  catastrophic on a non-admin caller path.
- Service-account keys are deliberately created with `user_id = None` at
  `key_management_endpoints.py`.
- The pre-fix endpoint code did `if user_id is None: user_id =
  user_api_key_dict.user_id`, then `if user_id != user_api_key_dict.user_id:
  raise 403`. With a service-account key, both sides are `None`, so the
  same-user check passes (`None != None` is `False`), and `entity_id =
  None` flowed straight into the unscoped global query — every tenant's
  daily spend, model breakdowns, and API key identifiers in one response.

The fix's correct shape is **"reject at the boundary, not at the value
site"**: `require_caller_user_id_for_non_admin` is called *before* any
fallback or comparison, so the bug class — `None` flowing through a
fallback into a builder that treats `None` as "no filter" — is closed at
the source and the same-user check at `:2592`/`:2690` becomes provably
non-vacuous.

The defense-in-depth `ValueError` at `ai_usage_chat.py:445-449` is the
right second line: even if a future endpoint copies the `_resolve_fetch_kwargs`
call site without remembering to call the entry guard, the unscoped-query
attempt fails loudly with a contributor-visible message naming the missing
guard. This is **belt-and-braces** done right — the second check exists to
catch *future* code, not to validate the current path.

The test trio is the strongest part of the diff. The
`mock_get_daily.assert_not_called()` line in both endpoint tests is the
load-bearing anti-behavior assertion: it pins not just "the endpoint
returns 403" but "the unscoped builder is never reached," which is the
actual security-relevant property. Without that assertion, a future
refactor that returned 403 *after* issuing the query (logging-then-fail)
would regress the leak silently.

The PR also correctly identifies the `is_admin` scoping branch is
unaffected (`/user/daily/activity?user_id=null` for an admin still means
"global view"), so the change has zero impact on legitimate admin global
queries.

## Backwards-compat note

PR body documents that service-account keys calling these three endpoints
will now get 403s instead of (incorrect) 200s. This is the right
fail-closed migration path — there is no "soft deprecation" for a
cross-tenant disclosure bug. The error-message text explicitly tells the
caller to use a user-bound key or proxy admin.

## Out-of-scope acknowledgment is correct

The PR body explicitly defers hardening the daily-activity builders
(`_build_where_conditions`, `_build_aggregated_sql_query`) to refuse
`entity_id=None` on non-admin paths to a follow-up PR — this is the right
call. That broader change touches all eight callers of `get_daily_activity`,
and conflating it with the surgical entry-guard fix would balloon the
review surface and delay the security patch.

## Verdict rationale

Right-shaped surgical security fix at the right boundary (entry-guard
helper called before any fallback), correct defense-in-depth tripwire at
the next-layer-down builder, and a strong test trio that pins both the
positive (helper returns user_id) and negative
(`assert_not_called()`/`raises HTTPException`) contracts end-to-end. The
admin-path-unaffected reasoning is sound, the backwards-compat plan is
the right fail-closed shape, and the deferred broader hardening is
correctly scoped out.

`merge-as-is`
