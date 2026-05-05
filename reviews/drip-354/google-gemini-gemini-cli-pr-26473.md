# google-gemini/gemini-cli PR #26473 — feat(cli): implement custom auth/status endpoint for Xcode ACP client

- Repo: `google-gemini/gemini-cli`
- PR: https://github.com/google-gemini/gemini-cli/pull/26473
- Head SHA: `0597443a4e51b52d20f936fb3d50356025f36290`
- Size: +263 / -0 across `packages/cli/src/acp/acpRpcDispatcher.ts`
  (impl) and `packages/cli/src/acp/acpRpcDispatcher.test.ts`
  (mocking + 7 new tests)

## Summary

Adds a new ACP `extMethod("auth/status")` to `GeminiAgent` for
Xcode ACP clients to query whether the current auth configuration
is actually usable, without having to attempt a real model call.
Implementation lives at
`packages/cli/src/acp/acpRpcDispatcher.ts:240-289`:

```ts
async extMethod(method: string, _params: unknown):
    Promise<Record<string, unknown>> {
  if (method === 'auth/status') {
    const clientName = this.context.config.getClientName()?.toLowerCase();
    const isXcode =
      clientName?.includes('xcode') || !!process.env['XCODE_VERSION_ACTUAL'];
    if (!isXcode) {
      throw new acp.RequestError(-32601, `Method not found: ${method}`);
    }
    return this.handleAuthStatus();
  }
  throw new acp.RequestError(-32601, `Method not found: ${method}`);
}
```

`handleAuthStatus()` (`acpRpcDispatcher.ts:251-289`) inspects the
content-generator's `authType` and returns
`{status: "Authorized" | "Unauthorized", methodId: string | null}`
for each of `USE_GEMINI`, `LOGIN_WITH_GOOGLE` (OAuth),
`USE_VERTEX_AI`, `GATEWAY`, and `COMPUTE_ADC`.

`checkOAuthValid()` (`acpRpcDispatcher.ts:291-322`) reads the
OAuth credentials JSON from `Storage.getOAuthCredsPath()`,
constructs an `OAuth2Client`, calls `getAccessToken()` then
`getTokenInfo()`, and returns true only if both succeed.

## Critical issue

**Hardcoded OAuth client_secret in the source tree**
(`acpRpcDispatcher.ts:308`):

```ts
const client = new OAuth2Client({
  clientId: '<682-prefixed-installed-app-client-id>',
  clientSecret: '<google-oauth-client-secret-literal-redacted>',
});
```

(Quoted shape only — the literal client_id and client_secret are
inlined in the diff at `acpRpcDispatcher.ts:305-308`.)

Two questions for the maintainers:

1. **Is this the same client_id/secret pair that the existing
   `LOGIN_WITH_GOOGLE` flow already uses elsewhere in the
   codebase?** If yes, this is duplicating known-public values
   (Google's OAuth-for-installed-apps doctrine treats the secret as
   "public" anyway) and the right fix is to *import* them from the
   existing constant location instead of re-pasting. If no, then
   this is introducing a new credential surface and the secret
   should not be in source.
2. **Either way, secret-scanning tools will flag this.** The
   value carries the well-known Google OAuth client-secret
   prefix and triggers GitHub secret scanning, gitleaks,
   trufflehog, etc. Even if the value is intentionally public for
   installed-app OAuth, the convention is to centralize it in one
   constants file with a comment explaining *why* it's safe to
   ship, not duplicate it inline in a feature module.

## Other concerns

1. **`extMethod` gating is inconsistent.** Lines 244-247 throw
   `Method not found` for *non-Xcode* clients, which makes the
   method effectively private to Xcode. But there's no
   `auth/status` capability advertised in the ACP handshake —
   well-behaved non-Xcode ACP clients won't even know to ask. The
   gating is defense-in-depth, fine, but the test
   `acpRpcDispatcher.test.ts:106-112` confirms a vscode client
   gets `Method not found: auth/status` rather than a proper
   "method exists but you're not allowed" — that's intentional per
   JSON-RPC convention to avoid leaking method existence.

2. **`isXcode` detection trusts an env var.** `process.env
   ['XCODE_VERSION_ACTUAL']` (line 244) means *any* process whose
   parent set that env var (e.g. a developer running CI scripts
   locally) will pass the gate. That's fine for "this is a non-
   secret status query" but would be unsafe to gate anything more
   sensitive on. Worth a comment that this is deliberately a low-
   trust gate.

3. **`checkOAuthValid` calls live network APIs**
   (`getAccessToken()`, `getTokenInfo()`, lines 312-318). On a
   slow/offline network the auth/status query will block until
   the underlying timeout. Consider a 5s `Promise.race` against a
   timeout so a UI poller can't get wedged.

4. **Test coverage is broad and well-mocked.** Seven scenarios
   covering: unknown method (line 99-103), non-Xcode rejection
   (105-112), missing API key (114-127), env API key (129-141),
   keychain API key (143-155), valid OAuth (157-174), invalid
   OAuth (176-192), Vertex AI (194-207). Each test mocks
   exactly the surface it depends on (mockGetAccessToken,
   mockGetTokenInfo, mockLoadApiKey, fs.readFile). This is
   genuinely good test hygiene.

5. **Mock leakage risk.** `vi.mock('node:fs', () => ({ promises:
   { readFile: vi.fn() } }))` (test:38-42) globally mocks
   `node:fs` for the whole test file. Anything else in the
   `GeminiAgent - RPC Dispatcher` describe block that touches
   filesystem will silently get `undefined`. Today only
   `auth/status` does, but a future test added higher up could
   hit a confusing failure. Scoping the mock under the
   `describe('extMethod - auth/status')` block via
   `vi.doMock` would isolate it.

6. **No test for `AuthType.GATEWAY` or `AuthType.COMPUTE_ADC`**
   branches (lines 277-283). Both have non-trivial logic; both
   should have one happy/sad test each.

## Verdict

**request-changes** — the hardcoded `clientSecret` at line 308
needs an answer before merge: either centralize into the existing
shared OAuth constants location with a comment, or remove and
load from configuration. Even if the value is public-by-design
for installed-app OAuth, duplicating it inline is a maintenance
hazard and triggers every secret-scanner in CI. The rest of the
PR (auth-status logic, test coverage, ACP method gating) is
solid; it's just blocked on that one line.
