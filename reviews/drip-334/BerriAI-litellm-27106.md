# BerriAI/litellm#27106 — fix(mcp): remove auth gate from OAuth broker authorize/token

- **Head SHA:** `8a7dda5a6f2bf3ca94514664eb5335fee9c99c12`
- **Author:** community
- **Size:** +764 / −29, 6 files (1 src, 3 tests, 2 e2e)

## Summary

A recent commit added `dependencies=[Depends(user_api_key_auth)]` to `/v1/mcp/server/oauth/{server_id}/authorize` and `/token`. Browsers can't send API keys to those URLs, so every browser-initiated OAuth flow returned 401. This PR removes the auth dependency on those two routes and reshapes `_get_cached_temporary_mcp_server_or_404` so it can handle anonymous callers safely while preserving allowlist semantics for authenticated non-admins.

## Specific citations

- `mcp_management_endpoints.py:1453` — signature change: `user_api_key_dict: Optional[UserAPIKeyAuth] = None`. Necessary so the helper can be called from now-anonymous endpoints.
- `mcp_management_endpoints.py:1475-1502` — rewritten access policy. Three branches:
  - `user_api_key_dict is None` + temp-cache server → allowed (browser flow against admin-created session).
  - `user_api_key_dict is None` + global-registry server → 403 with explicit "privilege inversion" rationale (`:1485-1495`). This is the security-critical part.
  - Authenticated non-admin: temp servers always denied (admin-internal); global servers require allowlist membership.
- `mcp_management_endpoints.py:1525-1530` — `mcp_authorize` no longer has `dependencies=[Depends(user_api_key_auth)]` and no longer takes `user_api_key_dict` parameter; passes only `(server_id, request=request)` into the helper.
- `mcp_management_endpoints.py:1568-1583` — same change for `mcp_token`.
- `tests/mcp_tests/test_mcp_oauth_security_unit.py` — new file, focused security regression tests.
- `tests/mcp_tests/test_mcp_oauth_flow_http_respx.py` — 417-line new file with end-to-end browser-flow tests, including `test_authorize_rejects_non_loopback_client_redirect_uri` (`:200-218` of diff) which guards the open-redirect surface that opens up alongside removing auth.

## Verdict: `merge-after-nits`

This is the right fix and the security analysis in the comment block at `:1475-1495` is exactly the reasoning a reviewer would want. The privilege-inversion concern is real and correctly addressed by 403'ing anonymous access to global-registry servers.

Nits / questions:

1. **Rate-limiting on anonymous endpoints.** Now that `/authorize` and `/token` accept anonymous callers, there's no per-key throttle. An attacker can enumerate `server_id`s without limits — even though they get 403 for global servers, the temp-cache lookup happens first and the response shape leaks whether a server_id exists in the temp cache. Recommend a low IP-based rate limit and constant-time response shape between "temp-cache miss" and "temp-cache hit but unauthorized".
2. **`server_id` enumeration via timing.** `get_cached_temporary_mcp_server(server_id)` then DB lookup; the timing difference between cache-hit and DB-miss could be observable. Low-priority, but worth noting.
3. **PR description says "remove auth gate" but the helper now does manual access control.** Title is mildly misleading — auth is removed from the *route decorator* but a hand-rolled three-branch policy replaces it. A title like "fix(mcp): allow anonymous OAuth broker access for browser flows; restrict to temp-cache servers" would track better.

Test coverage looks good. The respx integration test asserting `redirect_uri` rewrite (`:175-198` of diff) is exactly the kind of regression you want pinned.
