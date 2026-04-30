# BerriAI/litellm #26871 — fix(mcp): preserve oauth2 m2m auth for tools routes

- **Author:** Sameerlite (Sameer Kankute)
- **SHA:** `f2a455b`
- **State:** OPEN
- **Size:** +600 / -11 across 4 files
  (`litellm/proxy/_experimental/mcp_server/mcp_server_manager.py`,
  `.../mcp_server/server.py`, two new test files in
  `tests/test_litellm/proxy/_experimental/mcp_server/`)
- **Verdict:** `merge-after-nits`

## Summary

Closes a cross-trust auth gap in the MCP-server tools-call path where M2M (machine-to-machine)
OAuth2 servers were being authenticated with the caller-supplied `oauth2_headers`
rather than with the proxy-fetched client-credentials token. Two structural changes:
(1) introduces `MCPServerManager._resolve_oauth2_flow(...)` static helper at
`mcp_server_manager.py:170-200` that infers `oauth2_flow="client_credentials"` for
legacy DB rows that have `auth_type=oauth2 + token_url + client_id + client_secret`
but a null `oauth2_flow` field (correctly returns `None` if `authorization_url` is
present, since the presence of an authorization URL means interactive OAuth, not M2M);
threads this resolver through both the config-load path at `:375-382` and the
DB-build path at `:719-729`. (2) At `_call_regular_mcp_tool` at `:2484-2491` adds the
load-bearing gate: when `mcp_server.auth_type == MCPAuth.oauth2 AND
mcp_server.has_client_credentials`, sets `extra_headers = None` so the caller's
`oauth2_headers` (which would contain the user's interactive-OAuth bearer) cannot be
forwarded — instead the M2M token-fetch path supplies the upstream Authorization.
Backfilled with a 394-line `test_mcp_server.py` and a 100-line
`test_mcp_hook_extra_headers.py`.

## Reasoning

The auth-confusion bug is real — pre-fix, an M2M-configured MCP server upstream would
receive the user's interactive-OAuth bearer in the `Authorization` header (the user
having authenticated to the proxy via OAuth), which (a) leaks the user's bearer to
the upstream MCP server, and (b) breaks the M2M security model where the upstream
expects a service-account token tied to the proxy, not a user token. The
producer-side gate at `:2484-2491` is the right shape — explicit
`extra_headers = None` rather than relying on downstream code to ignore the field
later. The legacy-inference helper is the right migration discipline: the
denial-of-default `if oauth2_flow:` check at `:188-190` correctly refuses to
auto-promote unknown values, and the `if authorization_url: return None` at
`:194-195` correctly distinguishes interactive flow from M2M. Three nits worth
addressing: (1) the helper is a `@staticmethod` accepting 6 keyword-only params and
called from two sites with near-identical kwarg shapes — a small `OAuth2ResolveInputs`
NamedTuple/dataclass would make the call sites readable and the helper signature
stable; (2) the `# noqa: PLR0915` comment added to `_call_regular_mcp_tool` at
`:2421` is treating-the-symptom — that function was already long enough to trip
"too many statements" and adding more without an extraction is technical debt; (3)
the `has_client_credentials` predicate isn't visible in the diff slice — verify on
the maintainer side that it correctly returns False for `authorization_code` flow
(since both flows can have `client_id+client_secret` populated in the DB row, the
predicate must gate on flow type, not just on the presence of the credentials). The
494-line test surface is healthy coverage for a security-relevant change.
