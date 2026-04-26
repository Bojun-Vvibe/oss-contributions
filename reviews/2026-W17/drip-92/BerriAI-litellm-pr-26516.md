---
pr: 26516
repo: BerriAI/litellm
sha: 91f6661b37e88e60ff19d848e0c0edbba8d8423c
verdict: merge-as-is
date: 2026-04-27
---

# BerriAI/litellm #26516 — [Fix] Align MCP OAuth proxy endpoints with per-server access policy

- **Author**: yuneng-berri
- **Head SHA**: 91f6661b37e88e60ff19d848e0c0edbba8d8423c
- **Size**: +164/-24 across 2 files (1 endpoint file, 1 test file).

## Scope

Three MCP OAuth proxy endpoints — `/server/oauth/{server_id}/authorize`, `/token`, `/register` — currently authenticate the caller but **do not** check that the caller has access to the requested `server_id` before resolving the server record from the global registry and using its stored credentials to drive the OAuth exchange. Anyone with a valid proxy key can drive an OAuth handshake against any server in the registry. This PR moves the per-caller access check into the shared `_get_cached_temporary_mcp_server_or_404` helper so all three endpoints inherit the policy already enforced by `fetch_mcp_server`.

## Specific findings

- `mcp_management_endpoints.py:1448-1455` — signature change: `_get_cached_temporary_mcp_server_or_404(server_id, request=None)` becomes `(server_id, user_api_key_dict: UserAPIKeyAuth, request=None)`. Caller-required, can't be defaulted — exactly right for an authz check (you must not be able to "forget" to pass the auth).
- `:1456` — `resolved_from_temp_cache = server is not None` flag set before the fallback path. This is the key insight for the policy: if a server was resolved from the **temp cache**, it came from the admin-only `/server/oauth/session` flow and is by construction not exposed to non-admins (`:1480-1485` comment). For non-admins, the fallback path through `global_mcp_server_manager.get_mcp_server_by_id` is the only legitimate route, and that requires the server to be in their allowed-set.
- `:1473-1493` — the policy block. For admin callers (`_user_has_admin_view`), no restriction. For non-admins:
  - `if resolved_from_temp_cache: 403` — temp-cache servers are admin-only, period.
  - `else: get_allowed_mcp_servers(user_api_key_dict)` and `403` if `server.server_id not in allowed_server_ids`.
  Both 403s use a generic "Access denied to MCP server {server_id}" message — doesn't leak whether the server exists, good.
- `:1515,:1561,:1599` — three call-sites updated to pass `user_api_key_dict` (already in scope at each endpoint via `Depends(user_api_key_auth)`). Three-line mechanical change once the helper signature is fixed.
- The fallback import of `global_mcp_server_manager` was previously inlined inside `_get_cached_temporary_mcp_server_or_404` (`:1456-1458` of the old version) — now hoisted to module scope (`:1457-1459` removed in diff). Implies module-scope import was added elsewhere; assuming so since the file presumably parses, but worth a sanity check that the hoist was actually made and isn't relying on a re-export.
- Tests: `test_mcp_management_endpoints.py` adds `test_get_cached_temporary_mcp_server_non_admin_denied` and `test_get_cached_temporary_mcp_server_non_admin_allowed` (both at `:1359+`), exercising the two non-admin branches with `LitellmUserRoles.INTERNAL_USER` + `AsyncMock(return_value=[...])` for the allowed-set query. Updates the existing admin test to thread `admin_auth` through. Coverage matches the policy surface.
- Missing test: temp-cache + non-admin (the `resolved_from_temp_cache: 403` branch at `:1481-1485`). The existing tests only cover the registry-fallback path. Should add a one-liner.

## Risk

Low for the fix (additive-restrictive — strictly tightens access, no caller breaks who was previously authorized). Pre-fix this is a real authz bug: a leaked or low-privilege proxy key could initiate OAuth handshakes against the proxy's stored credentials for any registered MCP server. Worth a CVE-style note in release notes.

## Verdict

**merge-as-is** — clean centralization of authz into the shared helper, signature change forces compile-time discovery of any missed call-sites, tests cover both non-admin branches that matter most. Add the temp-cache + non-admin test in a follow-up; not worth blocking. Should be released with a security advisory acknowledging the prior gap.
