# BerriAI/litellm PR #27027 — fix(proxy): redact MCP server URL and headers for non-admin viewers (VERIA-8)

- PR: https://github.com/BerriAI/litellm/pull/27027
- Head SHA: `4ea32d13c911138e05c3d96e5ac4289380ec800a`
- Author: @stuxf

## Summary

Many MCP integrations (e.g. Zapier) embed an upstream API key directly in the server URL — `https://actions.zapier.com/mcp/<api-key>/sse`. The `/v1/mcp/server` and `/v1/mcp/server/{id}` endpoints previously returned the raw URL to any authenticated user; `_redact_mcp_credentials` only stripped the explicit `credentials` field and `_sanitize_mcp_server_for_virtual_key` only ran for restricted virtual keys. A non-admin internal user could call the API (or click the dashboard's unmask toggle) and exfiltrate the bearer token. This PR adds `_sanitize_mcp_server_for_non_admin` that runs on top of credential redaction and clears `url`, `spec_path`, `static_headers`, `extra_headers`, `env`, `authorization_url`, `token_url`, `registration_url` while preserving identity fields (`server_id`, `alias`, `mcp_info`). Frontend hides the unmask toggle for non-admins as defense-in-depth.

## Specific references from the diff

- `litellm/proxy/management_endpoints/mcp_management_endpoints.py:480-512` — new `_sanitize_mcp_server_for_non_admin` that calls `_redact_mcp_credentials` then nulls/empties the eight credential-bearing fields with values matching each field's declared default on `LiteLLM_MCPServerTable` (`None` for Optional, `[]`/`{}` for required collections).
- `:514-517` — list-helper `_sanitize_mcp_server_list_for_non_admin`.
- `:960-964` — wired into `fetch_all_mcp_servers` after the existing virtual-key branch and gated on `not _user_has_admin_view(user_api_key_dict)`.
- `:1333-1336` — same wiring for `fetch_mcp_server` (single-server path).
- Test file `tests/test_litellm/proxy/management_endpoints/test_mcp_management_endpoints.py`: existing `test_list_mcp_servers_non_admin_user_filtered` updated to assert `server.url is None` for all non-admin returns, plus three new tests at the bottom — `test_list_mcp_servers_non_admin_url_redacted` (full E2E with mocked manager), `test_list_mcp_servers_admin_keeps_url` (positive control for `PROXY_ADMIN`), and `test_sanitize_mcp_server_for_non_admin_clears_credential_fields` (direct unit test on the helper).
- UI: `ui/litellm-dashboard/src/components/mcp_tools/mcp_server_view.tsx:147-158` — adds `&& isProxyAdmin` to the toggle render condition with an inline comment explaining the defense-in-depth intent.

## Verdict: `merge-as-is`

This is a clean security fix: the threat model is named explicitly (URL-embedded bearer tokens), the patch closes both the API leak and the UI leak, the field reset values are correct against the model's defaults, and the test suite has positive (admin-keeps-URL), negative (non-admin gets stripped), and unit-level coverage. The existing `_sanitize_mcp_server_for_virtual_key` path is untouched, preserving the stricter behavior for restricted virtual keys.

## Nits / concerns

1. **Audit other non-admin read paths.** This PR fixes the two `/v1/mcp/server` endpoints, but if any other endpoint (e.g. team-scoped listings, OpenAPI introspection routes that surface `spec_path`, or batch fetchers used by the dashboard's "model dependencies" view) returns `LiteLLM_MCPServerTable` to a non-admin caller without going through these helpers, the same leak exists there. Worth a follow-up grep for `LiteLLM_MCPServerTable` returns and a short note in the PR body confirming the audit happened.
2. **Reset values vs response schema.** `extra_headers = []` and `env = {}` are correct for the SQL model defaults, but the response model that serializes back to JSON may emit `extra_headers: []` where it previously emitted the populated list — fine for security, but a downstream client parsing those fields will silently get an empty list rather than a "redacted" sentinel. Consider whether the dashboard wants a separate "redacted" indicator so users understand they're seeing a sanitized view.
