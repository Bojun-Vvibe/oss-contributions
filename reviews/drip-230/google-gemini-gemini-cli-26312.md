# google-gemini/gemini-cli #26312 — fix(core): refresh MCP OAuth token usage after re-auth

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26312
- **Head SHA**: `3a9b8a9d4418ed6db3b2250bd447fc0240483d76`
- **Files reviewed**: `packages/core/src/tools/mcp-client.ts`, `packages/core/src/tools/mcp-client.test.ts`
- **Date**: 2026-05-01 (drip-230)

## Context

The MCP HTTP/streamable-HTTP transport in `mcp-client.ts` previously resolved an OAuth
access token **at transport creation time** and wrote it into the static
`headers['Authorization'] = 'Bearer ${accessToken}'` slot of `StreamableHTTPClientTransport`
options. That meant: if the token expired during a long-running CLI session and
`MCPOAuthProvider` did a background refresh into `MCPOAuthTokenStorage`, the live
transport instance kept sending the old (now-rejected) token until process restart.
Issue #18895 is the user-facing report.

The PR makes token retrieval **dynamic** by introducing an `McpAuthProvider`
implementation that exposes a `tokens()` method the MCP SDK calls per request, instead of
a static `Authorization` header.

## Diff

New class at `mcp-client.ts:1035-1083`:

```ts
class DynamicStoredOAuthProvider implements McpAuthProvider {
  readonly redirectUrl = '';
  readonly clientMetadata: OAuthClientMetadata = {
    client_name: 'Gemini CLI (Stored OAuth)',
    redirect_uris: [],
    grant_types: [],
    response_types: [],
    token_endpoint_auth_method: 'none',
  };

  private clientInfo?: OAuthClientInformation;
  private readonly tokenStorage = new MCPOAuthTokenStorage();
  private readonly oauthProvider = new MCPOAuthProvider(this.tokenStorage);

  constructor(private serverName: string, private serverConfig: MCPServerConfig) {}

  async tokens(): Promise<OAuthTokens | undefined> {
    if (this.serverConfig.oauth?.enabled && this.serverConfig.oauth) {
      const token = await this.oauthProvider.getValidToken(this.serverName, this.serverConfig.oauth);
      if (!token) return undefined;
      return { access_token: token, token_type: 'Bearer' };
    }
    const credentials = await this.tokenStorage.getCredentials(this.serverName);
    if (!credentials) return undefined;
    const token = await this.oauthProvider.getValidToken(this.serverName, { clientId: credentials.clientId });
    if (!token) return undefined;
    return { access_token: token, token_type: 'Bearer' };
  }

  saveTokens(_tokens: OAuthTokens): void {}
  redirectToAuthorization(_authorizationUrl: URL): void {}
  saveCodeVerifier(_codeVerifier: string): void {}
  codeVerifier(): string { return ''; }
}
```

Wiring at `mcp-client.ts:2261-2305` (the `httpUrl || url` branch):

```diff
- let accessToken: string | null = null;
+ let shouldUseDynamicOAuthProvider = false;
  if (mcpServerConfig.oauth?.enabled && mcpServerConfig.oauth) {
+     shouldUseDynamicOAuthProvider = true;
      // ...still does an early validity probe via getValidToken to surface auth failures...
  } else {
-     accessToken = await getStoredOAuthToken(mcpServerName);
+     const tokenStorage = new MCPOAuthTokenStorage();
+     const credentials = await tokenStorage.getCredentials(mcpServerName);
+     shouldUseDynamicOAuthProvider = !!credentials;
  }
- if (accessToken) {
-     headers['Authorization'] = `Bearer ${accessToken}`;
+ if (shouldUseDynamicOAuthProvider) {
+     authProvider = createDynamicOAuthTokenProvider(mcpServerName, mcpServerConfig);
  }
```

## Observations

1. **Correct fix at the contract boundary.** The MCP SDK's
   `StreamableHTTPClientTransport` accepts an `authProvider` whose `tokens()` is invoked
   on each authenticated request; that's the SDK-blessed path for dynamic credentials.
   The PR uses it instead of stamping a one-shot header at construction time. That's the
   right shape — no need to recreate the transport on token refresh.

2. **Two arms preserved.** The previous code distinguished
   `oauth.enabled` config (explicit OAuth setup) from "we have stored tokens from a
   prior interactive flow". Both arms now route to the same
   `DynamicStoredOAuthProvider`, which internally branches on `serverConfig.oauth?.enabled`
   to pick between calling `getValidToken(serverName, serverConfig.oauth)` and
   `getValidToken(serverName, { clientId: credentials.clientId })`. The branch is
   preserved, so server config without `oauth.enabled` but with stored credentials still
   works.

3. **`saveTokens` / `redirectToAuthorization` / `saveCodeVerifier` are deliberate
   no-ops.** The `DynamicStoredOAuthProvider` is *consumption-only* — refresh happens
   inside `MCPOAuthProvider.getValidToken` which writes back into `MCPOAuthTokenStorage`
   directly, so the SDK's `saveTokens` callback would be a redundant second write. Same
   reasoning for `saveCodeVerifier` (stored elsewhere) and `redirectToAuthorization`
   (re-auth flow is a separate explicit user action, not an automatic SDK redirect for
   long-running tools). Comments documenting *why* each is a no-op would help the next
   reader; right now it looks like an empty stub.

4. **Test coverage is exactly what's needed.** Both arms get a test:
   - `mcp-client.test.ts:1780-1804` — `oauth.enabled: true` server, asserts
     `_authProvider` is set and that calling `_authProvider.tokens()` returns
     `{ access_token: 'fresh-token' }` from the mocked `getValidToken`. That's the load-bearing
     property: dynamic, not static.
   - `:1806-1841` — stored-token arm with no explicit `oauth.enabled`, same shape,
     returns `'stored-fresh-token'`.

   Both directly observe the `_authProvider.tokens()` round trip rather than asserting on
   `_requestInit.headers.Authorization`, which is the correct migration of the test
   contract.

## Risks / nits

- **`MCPOAuthTokenStorage()` and `MCPOAuthProvider(this.tokenStorage)` are constructed
  per-`DynamicStoredOAuthProvider` instance.** That's one storage handle per server
  config per transport creation. If `MCPOAuthTokenStorage` opens a file on construction
  (it likely does — it's the persistent store), each transport creation now does an
  extra open. Worth confirming the storage class has a singleton or pool so this
  doesn't bloat fd usage with many MCP servers configured.
- **Two `MCPOAuthTokenStorage()` instantiations exist in the same flow** — one inside
  `DynamicStoredOAuthProvider` at `:1047` and one inside `createTransport` at
  `:2287-2289` to do the credentials existence probe. They could share — pass the
  outer `tokenStorage` into the constructor — to avoid the double-open.
- **`saveTokens(_tokens: OAuthTokens): void {}`** silently dropping refresh-callback
  data is correct *given* the architecture (refresh happens upstream), but if the SDK
  later starts using `saveTokens` to communicate **rotated** refresh tokens that
  `MCPOAuthProvider` doesn't know about, this would silently lose them. A defensive
  TODO/log here would future-proof the contract.
- **No test for the cross-request refresh scenario itself** — i.e., call `tokens()`
  twice, return token A first and token B second. The current tests assert *dynamic
  lookup happens*; they don't assert *the dynamic lookup actually picks up a refresh
  between calls*. Adding one more arm where the mock `getValidToken` returns different
  values per call would pin the actual #18895 fix invariant.

## Verdict

merge-after-nits
