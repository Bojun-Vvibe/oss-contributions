# BerriAI/litellm PR #27190 — fix: replace user api key auth with authorization or cookie for mcp server creation

- URL: https://github.com/BerriAI/litellm/pull/27190
- Head SHA: `a1cda015b6b28e5745ed390a6487deaf88e336fd`
- Author: dennishenry
- Size: +153 / -6 (2 files: `mcp_management_endpoints.py`, `test_mcp_management_endpoints.py`)

## Verdict
`merge-after-nits`

## Rationale

This fixes a real regression introduced by commit `9deefc0f76`, which gated
`/v1/mcp/server/oauth/{server_id}/authorize` and `/.../token` behind
`dependencies=[Depends(user_api_key_auth)]`. Both endpoints are reached via browser-driven OAuth
flows — `/authorize` via `window.location.href` redirect (no opportunity to set headers) and
`/token` via `fetch()` from a context where the SPA has only the SSO session cookie, not the raw
master/user key. With `master_key` set, the standard `user_api_key_auth` raises 401, breaking the
MCP browser OAuth handshake end-to-end. The fix at
`mcp_management_endpoints.py:1501-1547` adds a per-endpoint `_mcp_oauth_user_api_key_auth`
dependency that tries the `Authorization` header first (preserving non-browser callers and
header-priority precedence) and falls back to decoding the `token` cookie JWT signed with
`master_key`, lifting the API key out of the standard SSO/username-password session payload —
mirroring the existing `byok_oauth_endpoints.py:104-124` cookie-decode pattern, so this is
established proxy convention rather than a new auth surface.

Test coverage at `test_mcp_management_endpoints.py:1556-1647` pins both branches of the conditional:
`test_mcp_oauth_user_api_key_auth_falls_back_to_token_cookie` constructs a real
`jwt.encode({"login_method": "sso", "key": "sk-test-cookie-key", ...}, master_key, "HS256")`,
asserts the call into `_user_api_key_auth_builder` carries `api_key="Bearer sk-test-cookie-key"`,
and `..._uses_authorization_header_when_present` pins that an explicit `Authorization` header wins
over a populated cookie. Both endpoints are correctly switched at `:1600` and `:1645`
(both the `dependencies=[…]` *and* the typed `user_api_key_dict: UserAPIKeyAuth = Depends(…)`
parameter — getting only one would have left a mixed-auth foot-gun). The per-server access control
added by `9deefc0f76` is preserved because `_mcp_oauth_user_api_key_auth` returns the same
`UserAPIKeyAuth` object the standard path returns; non-admins still hit the temp-cache server check
downstream.

Nits to fix before merge: (1) `options={"verify_exp": False}` at `:1525` silently accepts forever-valid
session cookies — comment says "UI session cookies may omit exp", which is true, but the pattern
should be `options={"verify_exp": "exp" in jwt.get_unverified_claims(token_cookie)}` so cookies
that *do* carry `exp` still get expiry-checked; right now a never-expiring SSO cookie is just as
acceptable as a short-lived one. (2) `_jwt.InvalidTokenError` at `:1530` swallows *all* JWT errors
silently (signature mismatch, malformed payload, wrong algorithm) — in production this means a
forged or replayed cookie indistinguishably falls back to the empty-`api_key` path which then 401s
on `_user_api_key_auth_builder`, costing the operator a debuggable signal; one
`verbose_proxy_logger.debug` line on the exception would make on-call life materially easier.
(3) `algorithms=["HS256"]` at `:1523` is hardcoded — `byok_oauth_endpoints.py` reads
`master_key_signing_algorithm` from config; using the same source here would prevent drift if the
proxy ever rotates to RS256.
