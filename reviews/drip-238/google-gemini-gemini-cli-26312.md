# google-gemini/gemini-cli#26312 — fix(core): refresh MCP OAuth token usage after re-auth

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26312
- **Author**: sahilkirad
- **Head SHA**: `3a9b8a9d4418ed6db3b2250bd447fc0240483d76`
- **Files (top)**: `packages/core/src/tools/mcp-client.ts`, `packages/core/src/tools/mcp-client.test.ts`
- **Verdict**: **merge-after-nits**

## Context

Pre-fix `createTransport` at `mcp-client.ts:2189-2229` (for `httpUrl`/`url` MCP servers) resolved an OAuth access token *once* at transport creation, baked it into the static `Authorization: Bearer <token>` header, and handed the transport off. When that token expired, the transport kept sending the stale `Bearer` header — there was no re-authentication path because the header was frozen at construction time. Users had to disconnect/reconnect the MCP server manually to pick up a refreshed token.

## What's right

- **`DynamicStoredOAuthProvider` class at `:1035-1090` implements the MCP SDK's `McpAuthProvider` interface with `tokens()` resolving lazily.** The SDK's `StreamableHTTPClientTransport` calls `_authProvider.tokens()` on every request when an auth provider is installed, so the token lookup happens at request-time rather than transport-construction-time. This is the structurally correct fix — the closure-over-stale-token pattern (very similar to opencode #25135 reviewed in drip-237) closes by routing all token reads through the resolver.
- **`tokens()` at `:1062-1080` correctly handles both OAuth-config branches.** First branch (`serverConfig.oauth?.enabled`): full OAuth config present, call `oauthProvider.getValidToken(serverName, serverConfig.oauth)` which runs the refresh-if-needed path. Second branch: only stored credentials present (from prior interactive auth), call `getValidToken(serverName, {clientId: credentials.clientId})` with the synthesized minimal-config. Both paths return `{access_token, token_type: "Bearer"}` so the transport sees a uniform `OAuthTokens` shape.
- **`createTransport` flip at `:2261-2304`** changes from "resolve token + bake into headers" to "decide whether to use dynamic provider" via `shouldUseDynamicOAuthProvider: bool`. The decision is driven by config presence (`serverConfig.oauth?.enabled` → true) or stored-credentials presence (`tokenStorage.getCredentials(...)` returns truthy → true). Then at `:2298-2304` if `shouldUseDynamicOAuthProvider`, install the dynamic provider; the SDK takes over from there.
- **Test rewrite at `mcp-client.test.ts:1777-1847`** flips both `createTransport`-via-`httpUrl` test cases from "asserts `_url` and `_requestInit.headers`" to "asserts `_authProvider` is defined and its `.tokens()` returns the freshly-resolved token". Correctly uses `vi.mocked(MCPOAuthProvider).mockReturnValue({getValidToken: mockGetValidToken})` to stub the provider, then asserts the token plumbed through. The two scenarios (OAuth-enabled config + stored-credentials-only) cover both branches in `tokens()`.

## Risks / nits

- **`DynamicStoredOAuthProvider` instantiates `MCPOAuthTokenStorage` and `MCPOAuthProvider` per-instance** at `:1052-1053`. If a single host has 10 MCP servers with stored OAuth credentials, that's 10 token-storage clients and 10 provider instances. Probably fine (these are likely lightweight) but worth confirming `MCPOAuthTokenStorage`'s constructor doesn't open a file handle or DB connection per instance.
- **The four no-op stubs at `:1082-1088` (`saveTokens`, `redirectToAuthorization`, `saveCodeVerifier`, `codeVerifier`) silently swallow flow steps that could matter** if the SDK's OAuth flow ever wants to drive the auth-code dance through this provider. Pre-fix the stored-credentials path was passive (just inject Bearer header); post-fix the SDK now believes there's an auth provider and may call `redirectToAuthorization` on a 401 — which silently no-ops, leaving the user with an opaque failure. Recommend at least `verbose_logger.warn("Dynamic OAuth provider received SDK auth-flow callback X; this path is unsupported, please re-authenticate via /mcp")` on each stub so a misbehaving server's 401 produces actionable diagnostics.
- **`clientMetadata` is a hardcoded literal at `:1037-1043` with empty `redirect_uris`/`grant_types`/`response_types`**. If the SDK's auth flow ever validates these (e.g. for dynamic client registration), the empty arrays would fail validation in non-actionable ways. A `// intentionally empty: this provider only serves stored tokens; full OAuth flow is handled elsewhere` comment would pin the intent.
- **Both new tests assert `testableTransport._authProvider` via type assertion to a private-leading-underscore field**. Brittle against MCP SDK internal renames — if the SDK ever moves to `_auth` or `authProvider`, the tests silently pass on the assertion-form but assert nothing meaningful. Consider asserting the *behavior* (issue a request through the transport, observe the Authorization header carries the fresh token) rather than the *implementation detail* (a private field exists).
- **The `getRequestHeaders?.()` call at `:2264` runs against the result of `createAuthProvider(mcpServerConfig)` — but the new `DynamicStoredOAuthProvider` doesn't implement `getRequestHeaders`.** If a future code path inspects `headers` collected from `getRequestHeaders` *before* the SDK gets a chance to call `tokens()`, that path would see an empty headers dict. Today this seems to only feed the `headers` variable used elsewhere in the function but worth confirming no consumer reads it expecting the Bearer to already be there.

## Verdict

**merge-after-nits.** Structurally correct fix at the right boundary (resolver function instead of frozen value, parallel to opencode #25135's MCP-reconnect pattern). Strongest nits: the four silent no-op stubs (`saveTokens` etc.) need at least a warn-log so opaque failures become diagnosable, and the test should assert behavior over private field shape.
