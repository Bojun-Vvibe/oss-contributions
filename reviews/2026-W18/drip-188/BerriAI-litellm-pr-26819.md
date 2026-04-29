# BerriAI/litellm#26819 — chore(team): require team-management role on /team/{id}/callback endpoints

- PR: https://github.com/BerriAI/litellm/pull/26819
- HEAD: `6c386af`
- Author: stuxf
- Files changed: 4 (+417 / -9)

## Summary

Closes a cluster of IDOR (insecure direct object reference) holes on
team-callback and organization-management endpoints by routing every
mutating handler through the existing `_verify_team_access` /
`_verify_org_access` guards. Pre-fix, any authenticated key holder could
`POST /team/{id}/callback` to overwrite Langfuse / Langsmith / GCS
credentials on any other team — and read them back via `GET` since the
endpoint serializes the metadata dict. Adjacent organization-member
mutation endpoints had the same shape (docstrings claimed "proxy_admin or
org_admin only" but enforcement was absent). Comprehensive regression
tests added.

## Cited hunks

- `litellm/proxy/management_endpoints/team_callback_endpoints.py:16-22,
  102-109, 207-214, 322-329` — four call sites
  (`add_team_callbacks`, `disable_team_logging`, the
  re-enable path, `get_team_callbacks`) all gated through
  `_verify_team_access(team_obj=LiteLLM_TeamTable(**_existing_team.model_dump()),
  user_api_key_dict=user_api_key_dict)`. The model_dump round-trip is a
  small inefficiency but matches the pre-existing helper signature;
  worth a follow-up to accept the raw row directly.
- `litellm/proxy/management_endpoints/team_callback_endpoints.py:251-259,
  338-345` — new `except HTTPException: raise` / `except ProxyException:
  raise` branches *before* the catch-all `except Exception`, so legitimate
  4xx responses from the access guard (403) and not-found checks (404)
  no longer get rewrapped into a 500 with the misleading "Internal
  Server Error" message + error-level log noise. Important: this changes
  observable status codes from "always 500" to "403/404 honestly," which
  may affect monitoring dashboards alerting on the old shape. Worth a
  CHANGELOG note.
- `litellm/proxy/management_endpoints/organization_endpoints.py:503-516`
  — `update_organization` gains a missing-`organization_id` 400 check
  *and* a `_verify_org_access` call. Without this any authenticated key
  holder could rewrite another org's metadata, budgets, and object
  permissions.
- `litellm/proxy/management_endpoints/organization_endpoints.py:927-935`
  (`organization_member_add`), `:1046-1053` (`organization_member_update`),
  `:1217-1223` (`organization_member_delete`) — same `_verify_org_access`
  gate added at the top of each member-mutation path. The
  `organization_member_update` comment correctly notes that the existing
  PROXY_ADMIN-target check covered "you can't promote someone *to*
  proxy_admin" but didn't cover "you have any permission to touch this
  org at all" — distinct properties.
- `tests/test_litellm/proxy/management_endpoints/test_organization_endpoints.py:568-700+`
  — comprehensive regression suite. Notable test design:
  `unauthorized_caller` fixture builds an `INTERNAL_USER` with no org
  memberships, `patched_org_prisma` mocks `find_unique` to return a
  victim row but `get_user_object` to return a caller with empty
  `organization_memberships`, and the four assertions all check
  `int(code) == 403` against either `HTTPException` or `ProxyException`
  (since the catch-all rewrap can convert 403 → ProxyException(code=403)).
  This is exactly the right test shape for an IDOR fix: assert the
  unauthorized caller hits a 403 *not* a 500.

## Risks

- The `LiteLLM_TeamTable(**_existing_team.model_dump())` round-trip is
  applied four times in `team_callback_endpoints.py`. If `model_dump()`
  produces a dict with extra fields the constructor rejects, this will
  raise at runtime instead of failing the access guard cleanly. A unit
  test that exercises a real `_existing_team` shape (not just a
  `MagicMock`) would catch that.
- Behaviour change: pre-fix, callers who hit a not-found team got a 500;
  post-fix, they get a 404. Rate-limiters or alerting that key off the
  old 500 shape need a heads-up. The PR doesn't include a CHANGELOG
  entry summarising the status-code surface changes.
- `_verify_team_access` is presumably already battle-tested by other
  team endpoints, but if it has any "is the caller an admin of this
  team's *parent* org" semantics that's now newly applied to callback
  endpoints; a maintainer who owns that helper should sanity-check the
  expected predicate matches the docstring intent.
- The test fixture `patched_org_prisma` patches at module path
  `"litellm.proxy.proxy_server.prisma_client"` rather than the
  `organization_endpoints` module. If a future refactor changes the
  import site, the mock will silently no-op and the test will pass
  trivially. Adding an `assert mock.find_unique.await_count == 1` to
  each test would make that breakage loud.
- No CVE / GHSA reference in the PR body even though the test docstring
  references "GHSA-xxv2-fprq-9x93 (team callback IDOR)". Worth linking
  in the PR description so downstream operators know to prioritize.

## Verdict

**merge-as-is**

## Recommendation

Land it; this is exactly the kind of access-control patch that earns
"merge-as-is" status — clear threat model, surgical guards routed
through existing helpers, generous regression tests asserting on the
correct status codes. Follow up the small nits (await-count assertion,
CHANGELOG entry, GHSA reference in PR description) in a non-blocking
PR.
