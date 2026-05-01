# BerriAI/litellm #26993 — feat(mcp): OAuth 2.0 Token Exchange (OBO) for MCP servers

- URL: https://github.com/BerriAI/litellm/pull/26993
- Head SHA: `4800446b2fe3cae1b9e4e9f22e1e50a539a399ad`
- Closes #21141 (refile)
- Files: 9 production + 415-line new test file. New `auth/token_exchange.py` (197 lines), changes to `mcp_server_manager.py`, `oauth2_token_cache.py`, `experimental_mcp_client/client.py`, types, constants.

## Context / problem

Pre-PR, an MCP server protected by an IDP couldn't accept the calling user's identity downstream — the proxy held its own OAuth2 client-credentials token (M2M) or a static configured token, and the MCP server only saw "the proxy" not "user X." This PR adds RFC 8693 Token Exchange: the user's JWT (`Authorization: Bearer <user-jwt>` arriving at the proxy) is exchanged at the IDP's token endpoint for a scoped token bound to (this user, this MCP server's audience), and that scoped token — not the raw user JWT — is forwarded to the MCP server. This is the canonical OBO ("on behalf of") shape and the right answer for per-user MCP authorization.

## Design analysis

### Type model — `types/mcp.py:38-45,120-138`
New `MCPAuth.oauth2_token_exchange` enum value, plus three optional credential fields on `MCPCredentials`:
```python
audience: Optional[str]
token_exchange_endpoint: Optional[str]
subject_token_type: Optional[str]  # default urn:...:access_token
```
The `MCPServer` model gains the same fields with a string default for `subject_token_type` and a typed `has_token_exchange_config` predicate at `mcp_server_manager.py:135-141`:
```python
return (
    self.auth_type == MCPAuth.oauth2_token_exchange
    and bool(self.client_id and self.client_secret)
    and bool(self.token_exchange_endpoint or self.token_url)
)
```
The predicate fail-closes on missing client credentials or endpoint — good. The `or self.token_url` fallback is documented (token_url reused as exchange endpoint when none set).

### Handler — `auth/token_exchange.py:1-197`
The `TokenExchangeHandler` class is the load-bearing piece:

1. **Cache** — `InMemoryCache(max_size_in_memory=MCP_TOKEN_EXCHANGE_CACHE_MAX_SIZE, default_ttl=...)` with key `sha256(f"{subject_token}:{server_id}")`. Hashing the user JWT means the cache key isn't itself a credential leak vector if dumped. Per-(user,server) granularity is correct: same user + different MCP servers get different scoped tokens (different audience, different scopes).
2. **Lock map** — `weakref.WeakValueDictionary[str, asyncio.Lock]` so concurrent first-request stampedes for the same `(user, server)` collapse to one IDP round-trip, and locks are GC'd when no coroutine holds a reference. This is the right shape for "many rotating user tokens" — a regular `dict` would leak per-user-token entries indefinitely.
3. **Double-checked locking** at `:81-91` (`cached = self._cache.get_cache(...)` outside the lock, then again inside) — standard pattern, correctly applied.
4. **TTL discipline** at `:158-162`:
   ```python
   ttl = max(expires_in - MCP_OAUTH2_TOKEN_EXPIRY_BUFFER_SECONDS, MCP_OAUTH2_TOKEN_CACHE_MIN_TTL)
   ```
   Cached TTL is the IDP's `expires_in` minus a buffer (so the proxy refreshes before the IDP rejects), floored at a minimum so a misconfigured IDP returning `expires_in: 1` doesn't cause cache thrash.
5. **Error mapping** at `:140-153` — IDP `HTTPStatusError` is logged with status + body at `verbose_logger.debug` then re-raised as `ValueError` with the server_id and status. The `verbose_logger.debug` (not `info` / `error`) for the IDP body is the right discipline: response bodies on token errors can contain the subject token or other sensitive context, debug-level is the right gate.
6. **Invalidation hook** at `:189-192` — `invalidate(subject_token, server_id)` for the "after a 401 from the MCP server, evict and retry" path. Not wired into the call site visibly in this diff — worth confirming the consumer path actually calls it (otherwise stale tokens from rotated IDP keys silently fail until TTL).

### Resolution chain — `oauth2_token_cache.py:264-285`
The `resolve_mcp_auth` priority is now:
```
1. mcp_auth_header (per-request override)
2. token exchange (when has_token_exchange_config && subject_token)
3. client_credentials cached
4. static authentication_token
```
Token exchange is correctly inserted between the per-request override and the M2M client-credentials cache — a per-request override should still win (operator escape hatch); a server configured for OBO with no incoming user token should fall through to client-credentials only if also configured for that, otherwise the bare server.authentication_token. The `if server.has_token_exchange_config and subject_token:` guard at `:281` means OBO is *only* attempted when both sides are present — no silent degradation to a less-scoped path when the operator wanted OBO but forgot the subject token.

### Subject token extraction — `mcp_server_manager.py:1163-1184`
```python
@staticmethod
def _extract_bearer_token(oauth2_headers, raw_headers) -> Optional[str]:
    auth_value = None
    if oauth2_headers and "Authorization" in oauth2_headers:
        auth_value = oauth2_headers["Authorization"]
    elif raw_headers:
        normalized = {k.lower(): v for k, v in raw_headers.items()}
        auth_value = normalized.get("authorization")
    if auth_value and auth_value.startswith("Bearer "):
        return auth_value[len("Bearer "):]
    return None
```
The two-source lookup (`oauth2_headers` first, then case-insensitively normalized `raw_headers`) is the right shape — ASGI servers vary on header case, and the `Authorization` header is case-insensitive on the wire. `Bearer ` (capital B + space) prefix check is correct. Returns `None` when no auth header — caller path at `:2589-2590` then leaves `subject_token=None` and `resolve_mcp_auth` falls through.

### Test coverage — `test_token_exchange.py:1-415`
22 tests covering: success path with RFC 8693 param assertion (verifies `grant_type`, `subject_token`, `subject_token_type`, `audience`, `scope`, `client_id`, `client_secret` all present in the POST body), audience/scope omission when None, cache hit (one HTTP call for two `exchange_token` invocations), error mapping, bearer-token extraction. The fixture-builder pattern (`_obo_server(**overrides)`, `_exchange_response(token, expires_in)`) is the right test ergonomics for a 22-case suite.

## Risks

- **`client_secret` storage and logging** — `client_secret` is on `MCPServer` as a plain string, posted in the form body to the IDP. Standard for client_secret_post auth, but worth confirming nothing logs the request body at debug level. The `verbose_logger.debug` call at `:128-133` logs server_id + endpoint + audience but not the data dict — good.
- **The `invalidate()` method is unused in this PR's diff** — without a 401-retry consumer, a rotated IDP signing key takes up to `MCP_OAUTH2_TOKEN_CACHE_DEFAULT_TTL` to recover. Worth either wiring the consumer in this PR or filing a follow-up issue.
- **Per-request override (`mcp_auth_header`) wins over OBO** — this is the documented order, but in an OBO deployment an operator who accidentally configures both could end up with an unscoped token forwarded. Worth a warning log at `resolve_mcp_auth` when `mcp_auth_header` AND `has_token_exchange_config` AND `subject_token` are all simultaneously present.
- **Cache key includes the full user JWT** before hashing — fine cryptographically (SHA-256), but the unhashed string is held in memory long enough for the hash. No leak, just noting.
- **`weakref.WeakValueDictionary` requires the lock be held by *something* outside the dict** — since `async with self._get_lock(cache_key):` holds a reference for the duration of the critical section, this works. Past the critical section the lock is GC-eligible. Correct shape.
- **No documentation PR linked** — operators need to know about the new `auth_type`, the new credential fields, and the failure modes. PR body has a good config example but proxy docs need an update before this is discoverable.
- **22 unit tests, no integration test** with a real IDP / live token exchange — reasonable for the unit-test layer; an integration test gated on a `RUN_LIVE_OBO_TESTS=1` env var would harden the contract against IDP shape drift.

## Suggestions

1. Wire `invalidate()` to the MCP-call 401 retry path (or file a follow-up).
2. Add the "operator-configured both override and OBO" warning log.
3. Add a docs PR to the same release.
4. Consider an integration test gated on env var.
5. Document `MCP_TOKEN_EXCHANGE_CACHE_MAX_SIZE` (default 500) and the buffer constants in the operator-facing config reference.

## Verdict

`needs-discussion` — the architecture is right (RFC 8693 OBO is the canonical shape for per-user MCP authorization, the type model + handler + resolution priority + subject-token extraction are all individually well-shaped, the cache + lock + TTL discipline are correct), but the surface is large (~720 lines + 415 lines test), the `invalidate()` hook isn't wired, the operator-configured-both case has no warning, and the docs PR is unlinked. Wants a focused security review on the IDP-error-body logging gate and a clear story for IDP key rotation before merge.
