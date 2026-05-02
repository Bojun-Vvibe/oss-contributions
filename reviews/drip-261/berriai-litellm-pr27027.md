# BerriAI/litellm PR #27027 — fix(proxy): redact MCP server URL and headers for non-admin viewers (VERIA-8)

- **Head SHA:** `4ea32d13c911138e05c3d96e5ac4289380ec800a`
- **Files:** 3 (`mcp_management_endpoints.py`, `test_mcp_management_endpoints.py`, `mcp_server_view.tsx`)
- **LOC:** +212 / −4

## Observations

- `mcp_management_endpoints.py:480-510` — `_sanitize_mcp_server_for_non_admin` strips the credential-bearing fields layered on top of `_redact_mcp_credentials`: `url`, `spec_path`, `static_headers`, `extra_headers`, `env`, `authorization_url`, `token_url`, `registration_url`. The threat model is correctly stated in the docstring — many MCP integrations embed the upstream API key in the URL path (`https://actions.zapier.com/mcp/<api-key>/sse`), so `url` redaction is the highest-impact change here.
- `mcp_management_endpoints.py:963-967` and `:1334-1336` — the new `if not _user_has_admin_view(user_api_key_dict)` branch is reached *after* the existing `is_restricted_virtual_key` branch, so virtual-key sanitization still wins for restricted keys. Order is correct: most-restrictive first, then admin gate, then default redaction. No regression for virtual keys.
- The default values for cleared fields match each field's declared default on `LiteLLM_MCPServerTable` (`None` for Optional, `[]`/`{}` for required list/dict) — this matters because many serializers and frontends rely on the field type, not nullability. Good attention to detail.
- `test_mcp_management_endpoints.py:2906-2960` — `test_list_mcp_servers_non_admin_url_redacted` constructs an INTERNAL_USER, plants a server with a token-bearing URL plus header/env/oauth secrets, and asserts every credential field is cleared while identity fields survive. The companion `test_list_mcp_servers_admin_keeps_url` confirms the admin path is unchanged. Strong negative + positive coverage.
- `test_mcp_management_endpoints.py:632-637` — the existing `test_list_mcp_servers_non_admin_user_filtered` was updated to assert `server.url is None` for the non-admin case. This is the diff that captures the actual behavior change and would catch any regression.
- `mcp_server_view.tsx:150-156` — frontend defense-in-depth: the URL toggle is now hidden for non-admins (`hasToken && isProxyAdmin`), with a comment explaining the backend already strips `url` but this is belt-and-suspenders. Sensible.

## Verdict: `merge-as-is`

- Threat model, code path, and test matrix all line up. The change is narrowly scoped and the field-clearing defaults are field-correct.
