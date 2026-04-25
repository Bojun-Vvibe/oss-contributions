# BerriAI/litellm#26463 — Tighten MCP public-route detection and OAuth2 fallback gating

- PR: https://github.com/BerriAI/litellm/pull/26463
- Author: stuxf
- +387 / -25
- Head SHA: `0a4640fbd0fd7729d276edb6b95d37fd02ec55c7`

## Summary

Two distinct security fixes in `MCPRequestHandler.process_mcp_request`,
filed against existing GHSA advisories (GHSA-7cwm-3279-qf3c and
GHSA-h8fm-g6wc-j228):

1. **Public-route bypass.** The previous check `".well-known" in
   str(request.url)` matched the marker anywhere in the full URL string —
   query, hostname, deeper segment — letting a caller smuggle the substring
   to bypass auth on any MCP route. Replaced with an exact path-prefix check
   on `request.url.path`.
2. **OAuth2 passthrough fallback was unconditional.** The 401/403 fallback
   added to support upstream OAuth2 MCP servers (e.g. Atlassian) replaced
   any failed LiteLLM auth with an anonymous `UserAPIKeyAuth()` regardless
   of the resolved target. Now gated to fire only when *every* resolved
   target server has `auth_type == oauth2`.

## Specific findings

- `litellm/proxy/_experimental/mcp_server/auth/user_api_key_auth_mcp.py:120`
  (SHA `0a4640fbd0fd7729d276edb6b95d37fd02ec55c7`) — replaces
  `if ".well-known" in str(request.url):` with `if
  request.url.path.startswith("/.well-known/"):`. Path-only and exact
  prefix; the regression test
  `test_well_known_substring_in_query_does_not_bypass_auth` at
  `test_user_api_key_auth_mcp.py:806` proves the query-string smuggling
  vector is closed, and `test_legitimate_well_known_path_still_bypasses_auth`
  at line 858 proves real OAuth metadata routes still work per RFC 8414/9728.
- `user_api_key_auth_mcp.py:147` — the merged
  `except (HTTPException, ProxyException) as e:` block replaces two near-
  duplicate `except` blocks. The status-code coercion handles
  `HTTPException.status_code` (int) and `ProxyException.code` (string,
  including the literal `"None"`) without raising on `int("None")`. The
  comment explicitly calls out this footgun — good defensive code.
- `user_api_key_auth_mcp.py:156` — fallback now requires
  `_target_servers_use_oauth2(path=request.url.path, mcp_servers=mcp_servers)`
  to be True. The helper at line 200 fails closed when:
  (a) the target list is empty, (b) any target lookup returns `None`, or
  (c) any target's `auth_type != MCPAuth.oauth2`. The "ALL targets must be
  OAuth2" rule (not "any") is the right call for a passthrough that
  consumes the bearer header — mixing one non-OAuth2 server in a
  multi-target request would otherwise leak the fallback.
- `user_api_key_auth_mcp.py:182` — `_extract_target_server_names_from_path`
  recognizes both `/mcp/{server_name}` and `/{server_name}/mcp` URL shapes
  and explicitly returns `[]` for anything else, so REST/admin endpoints
  fall through and the caller fails closed. Note: the doc-comment correctly
  flags that `x-mcp-servers` takes precedence — the call site at line 218
  honors that with `mcp_servers if mcp_servers is not None else
  _extract_target_server_names_from_path(path)`, which preserves the
  explicitly-empty-list-means-no-targets semantics.
- New test class `TestMCPOAuth2FallbackTargetGating` at line 1028 covers
  the three required cases: target is `api_key` → blocked (401 propagates),
  target unresolvable → blocked, target is `oauth2` → fallback allowed.
  Coverage is good.

## Verdict

`merge-as-is`

## Rationale

This is a clean security fix for two real bugs under existing GHSA IDs. Both
the path-prefix tightening and the per-target-OAuth2 gate fail closed in the
ambiguous cases (unresolvable target, missing `x-mcp-servers`, mixed target
list). Test coverage is thorough — both the smuggling vectors and the
legitimate paths are pinned down, and the existing positive tests are
updated to mock an `auth_type=oauth2` target so they continue to exercise
the supported passthrough path under the new contract. Nothing to nit.
