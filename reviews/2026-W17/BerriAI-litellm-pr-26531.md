# BerriAI/litellm #26531 — feat(mcp): OAuth2 self-service for internal users (LIT-2503)

- **Repo**: BerriAI/litellm
- **PR**: #26531
- **Author**: ishaan-berri
- **Head SHA**: 491bef28ffbb140d25624e87ca958200ddace8e8
- **Base**: main
- **Size**: +191 / −10 across the management endpoint
  (`mcp_management_endpoints.py`, +60/−5), three new dashboard
  components (`OAuth2ConnectButton.tsx`, +62; column glue in
  `mcp_server_columns.tsx`, +35; wiring in `mcp_servers.tsx`, +22),
  and three doc images.

## What it changes

Two server-side changes plus a UI affordance:

1. `fetch_all_mcp_servers` (`mcp_management_endpoints.py:907-925`)
   now also resolves `has_user_credential` for OAuth2 servers, not
   just BYOK servers. The `byok_server_ids` list is renamed
   `credentialed_server_ids` and accepts both
   `is_byok=True` and `auth_type == "oauth2"`. The post-query
   marking loop at `:923` mirrors the same `or` condition, so the
   set semantics stay symmetric.
2. The `/server/oauth/{server_id}/authorize` route gains a
   browser-redirect auth path. `_authorize_user_auth` at
   `mcp_management_endpoints.py:1503-1567` first tries
   `Authorization`/`x-litellm-api-key` headers via
   `_user_api_key_auth_builder`, then falls back to a `token`
   cookie verified with `pyjwt.decode(..., algorithms=["HS256"])`,
   and finally raises `401 login_required` if neither is present.
   The route's previous `dependencies=[Depends(user_api_key_auth)]`
   is removed so the new dependency is the sole gate.
3. UI: `OAuth2ConnectButton.tsx` renders a Connect / Connected /
   Reconnect button based on `server.has_user_credential`, and the
   columns layer (`mcp_server_columns.tsx:273-304`) routes OAuth2
   servers through it (with a Disconnect link wired to
   `deleteMCPOAuthUserCredential` in `mcp_servers.tsx:163-175`).

## Strengths

- The renamed list (`byok_server_ids` → `credentialed_server_ids`) and
  the matching `or auth_type == "oauth2"` predicate at `:911-913` and
  `:925-928` keep the two halves of the query/marking pair in lockstep —
  a future third credential type just needs to extend the predicate
  in both places.
- The fallback auth chain at `mcp_management_endpoints.py:1509-1530`
  has the right precedence: explicit header beats cookie, no header +
  no cookie returns 401 with a stable `login_required` detail string
  the UI can branch on.
- Good contract comment at `:1517-1519` explaining *why*
  `user_api_key_auth()` can't be invoked directly (its `Security()`
  params would be left as raw `Security` objects without FastAPI's
  DI), and explicitly calling `_user_api_key_auth_builder` instead.
- The internal-user filter in `mcp_servers.tsx:131-135` (only show
  approved/active servers to non-admins) is the right defensive
  posture — admins still see pending/disabled servers in the catalog,
  internal users don't.
- Dashboard reactivity: the `useMemo` for `columns` at `:178-194`
  now depends on `accessToken`, `handleOAuthDisconnect`, and `refetch`,
  so a token refresh or disconnect callback change won't leave stale
  column closures behind.

## Concerns / asks

- **Cookie-fallback security model is under-explained.** The cookie
  is verified against `master_key` with HS256, with no `iss`/`aud`
  validation, no expiry check beyond what pyjwt's default does, and
  no CSRF protection. The route is `GET /server/oauth/{server_id}/authorize`
  and it triggers a redirect to a third-party OAuth2 IdP — a
  cross-site request from an attacker page could potentially
  initiate a flow on behalf of the logged-in user. Worth either:
  - explicit `SameSite=Lax`/`Strict` requirement on the `token`
    cookie (and a comment in `_authorize_user_auth` asserting that),
  - or a CSRF token in the query string that's checked before the
    redirect is constructed.
  Even though the OAuth state parameter mitigates code-injection,
  it doesn't stop an unwanted flow from being kicked off.
- `pyjwt.decode(..., algorithms=["HS256"])` at `:1545` will silently
  accept a token whose alg header is *anything* HS256-compatible, but
  more importantly the algorithm whitelist should match what other
  parts of the proxy issue. If the dashboard ever switches to RS256 /
  asymmetric, this line will need updating in lockstep — worth
  extracting a `_dashboard_jwt_decode` helper used everywhere.
- `role_map` at `:1554-1559` defaults unknown roles to `INTERNAL_USER`.
  That's the right *availability* default, but for an authorize
  endpoint it's also the right *security* default to be more
  conservative — consider defaulting to `INTERNAL_USER_VIEW_ONLY`
  or rejecting unknown roles outright. A typo in the dashboard's
  cookie issuance code shouldn't silently grant write-equivalent
  capability.
- `OAuth2ConnectButton.tsx` and the inline copy in
  `mcp_server_columns.tsx:275-300` *both* render essentially the
  same Connected/Disconnect badge (the column inlines the
  has-credential branch and only delegates to the component for the
  not-connected branch). Pick one — either always delegate to the
  component (and pass `onDisconnect` through), or inline both.
  Two copies of "green badge with CheckOutlined" will drift.
- `redacted_mcp_servers` at `:907` — the comment further down the
  function mentions virtual keys get a sanitized discovery view; an
  inline comment near the new `or auth_type == "oauth2"` clauses
  noting "OAuth2 server URLs are public, only the per-user token is
  sensitive" would help future maintainers reason about why it's
  safe to expose `has_user_credential` here.

## Verdict

**needs-discussion** — the credentialing extension and the UI work
are clean, but the cookie-fallback auth path on a redirect endpoint
needs an explicit CSRF / SameSite story before merge. The role-map
default and the `pyjwt` algorithm-whitelist fragility are easier
fixes once the threat model is settled.

## What I learned

FastAPI's `Depends(...)` with `Security(...)` parameters is a DI-only
construct — calling the dependency function directly leaves the
`Security` objects unbound. The right pattern when you need to invoke
auth from inside another dependency is to call the underlying
*builder* (here `_user_api_key_auth_builder`) and pass the raw
header values yourself. That's a recurring sharp edge in FastAPI
codebases that mix request-scoped auth with branch-on-header /
fallback-to-cookie logic.
