# BerriAI/litellm #26993 — feat(mcp): OAuth 2.0 Token Exchange (OBO) for MCP servers

- **Head SHA:** `35a6801ccd3b0bdd6d0f810690870be5de6e0ed5`
- **Files:** `litellm/constants.py` (+5), `litellm/experimental_mcp_client/client.py` (+2), `litellm/proxy/_experimental/mcp_server/auth/token_exchange.py` (+197 NEW), `litellm/proxy/_experimental/mcp_server/mcp_server_manager.py` (+57 / -5), `litellm/proxy/_experimental/mcp_server/oauth2_token_cache.py` (+26 / -2), `litellm/types/mcp.py` (+18), `litellm/types/mcp_server/mcp_server_manager.py` (+13), `tests/.../test_token_exchange.py` (+415 NEW), plus 2 small test fixups
- **Verdict:** `merge-after-nits`

## Rationale

Implements RFC 8693 (OAuth 2.0 Token Exchange) so MCP servers can accept a user's incoming JWT and exchange it for a scoped upstream access token (on-behalf-of). Architecturally clean: the new `TokenExchangeHandler` in `token_exchange.py:33+` owns an `InMemoryCache(max_size=MCP_TOKEN_EXCHANGE_CACHE_MAX_SIZE)` keyed by `sha256(subject_token + server_id)` (line 67), uses a `weakref.WeakValueDictionary[str, asyncio.Lock]` to avoid unbounded lock growth across rotating user tokens (line 47), and applies the canonical double-check pattern around the lock (line 86-93) so the IDP only sees one exchange per (user, server) pair under load. The `_get_auth_headers` change in `client.py:368` correctly maps `MCPAuth.oauth2_token_exchange` to a `Bearer` header.

Nits / asks before merge: (1) cache key hashes the raw subject_token — `sha256` is fine but the verbose log paths must NEVER print `cache_key`-adjacent context that could make rainbow-style correlation easier; please audit `verbose_logger` calls for token leakage. (2) The `endpoint = server.token_exchange_endpoint or server.token_url` fallback in `_do_exchange` (line 119) is convenient but conflates two semantically distinct OAuth endpoints — at least a `verbose_logger.warning` when falling back would help operators notice misconfiguration. (3) 415 lines of new tests is great; please confirm at least one test exercises the IDP returning `expires_in` smaller than `MCP_OAUTH2_TOKEN_CACHE_MIN_TTL` to validate the floor. (4) `MCP_TOKEN_EXCHANGE_CACHE_MAX_SIZE` defaults to 500 — fine for small deployments but document the eviction story for high-cardinality user populations.

Net: substantive, well-isolated under `_experimental/`, with strong test coverage. Merge after the logging audit.

