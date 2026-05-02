# google-gemini/gemini-cli PR #26312 — fix(core): refresh MCP OAuth token usage after re-auth

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26312
- **Head SHA**: `3a9b8a9d4418ed6db3b2250bd447fc0240483d76`
- **Size**: +133 / −26 across 2 files (`packages/core/src/tools/mcp-client.ts`, `packages/core/src/tools/mcp-client.test.ts`)
- **Verdict**: **merge-after-nits**

## What changed

`mcp-client.ts` introduces `class DynamicStoredOAuthProvider implements McpAuthProvider` (lines ~1035–1097) and a `createDynamicOAuthTokenProvider(serverName, serverConfig): McpAuthProvider` factory (line ~1099). The provider's `tokens()` method is the live entry point: if `serverConfig.oauth?.enabled`, it calls `oauthProvider.getValidToken(serverName, serverConfig.oauth)`; otherwise it falls back to looking up stored credentials via `tokenStorage.getCredentials(serverName)` and asks `getValidToken(serverName, { clientId: credentials.clientId })`. Both paths return `{ access_token: token, token_type: 'Bearer' }` or `undefined`.

The transport-creation block in `createTransport` (lines ~2191–2305) is restructured. Previously the flow was: fetch a token *once*, jam it into a `Bearer …` header on the `headers` map, hand the frozen header dict to `StreamableHTTPClientTransport`. Now: when no static `authProvider` exists, decide `shouldUseDynamicOAuthProvider` by checking `oauth?.enabled` (true) or whether stored credentials exist (true), and if so call `createDynamicOAuthTokenProvider(...)` and let the transport's auth provider call `tokens()` per-request. The `Authorization` header bake-in path is removed.

Test changes (`mcp-client.test.ts`): the two old `should connect via httpUrl / without headers` and `with headers` tests are replaced with `uses MCP SDK authProvider token() path for oauth-enabled servers` and `uses dynamic authProvider when stored OAuth token exists`. Both stub `MCPOAuthProvider.getValidToken` to return a fresh token, then reach into the constructed transport via `(transport as unknown as { _authProvider?: { tokens: () => Promise<…> } })._authProvider!.tokens()` and assert `access_token` matches the stub.

## Why this matters

The previous flow froze the OAuth token at transport-creation time. After re-auth (token refreshed in storage), the running transport still carried the stale `Bearer` header in `_requestInit.headers`, and the SDK never re-read it. Result: every MCP call kept failing with 401 until the user restarted the CLI or recreated the transport. That's a real, painful, and silent UX bug for any long-lived MCP session against an OAuth-protected server. The fix is structurally correct: hand the SDK a provider object whose `tokens()` is called *at request time*, so a refreshed token flows through naturally.

## Specific call-outs

- `DynamicStoredOAuthProvider` constructs `new MCPOAuthTokenStorage()` and `new MCPOAuthProvider(this.tokenStorage)` as instance fields. **Nit**: this allocates a fresh storage + provider per transport. If `MCPOAuthTokenStorage` is keyed by file/keyring with any internal cache or lock, you may want to inject a shared instance via the factory rather than newing them up. At minimum, document that it's safe to create N of these per process.
- The `redirectToAuthorization`, `saveTokens`, `saveCodeVerifier`, `codeVerifier` methods are stubs returning empty/no-ops. **Nit**: this is fine for the stored-token path but means this provider explicitly cannot complete a fresh OAuth dance — the comment block should call that out so future readers don't try to reuse `DynamicStoredOAuthProvider` for first-time auth. Consider renaming the class to `StoredOnlyDynamicOAuthProvider` or adding `// Stored-token path only; first-time OAuth flow lives in createAuthProvider().` near the class doc.
- The `clientMetadata` block (lines ~1039–1045) declares empty `redirect_uris`, `grant_types`, `response_types`. That's consistent with the stubs above but again — only safe because this provider never actually drives a flow.
- The deleted tests previously asserted on `_url` and `_requestInit.headers` shape — internals of `StreamableHTTPClientTransport`. The new tests assert on `_authProvider.tokens()` — also internals. **Nit**: both styles depend on SDK private fields, but the previous shape would silently break if the SDK refactored, so this change at least re-anchors against the *correct* internal. Consider also adding an end-to-end test that mocks an HTTP server returning 401 → 200 across token refresh, to lock in the actual refresh behavior rather than just the wiring.
- The `let authProvider = createAuthProvider(...)` switch from `const` to `let` (line 2264) is a small but meaningful refactor — flag in commit message that re-binding is intentional.
- The branch around line 2294 still calls `tokenStorage.getCredentials(mcpServerName)` purely to set `shouldUseDynamicOAuthProvider = !!credentials`, throwing the credentials object away. The new provider will re-fetch them on first `tokens()` call. Two reads per startup is acceptable but worth noting; alternatively pass the credential through to avoid the second fetch.

## Verdict rationale

Real, well-targeted fix for a known re-auth-doesn't-take bug. The architectural change (frozen header → live provider) is correct. Three nits: clarify in code that this provider is stored-token-only and not for first-time flow, consider sharing storage instances, and add an HTTP-level integration test for the actual refresh round-trip. None blocking, all should ideally land before merge.
