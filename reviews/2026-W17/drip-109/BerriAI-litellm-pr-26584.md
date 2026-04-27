# BerriAI/litellm #26584 — [Feat] Add support for azure entra discovery endpoint

- **Repo**: BerriAI/litellm
- **PR**: #26584
- **Author**: Sameerlite
- **Head SHA**: 3e4f9af9555df6fef78e22624778e655d3067173
- **Size**: +167 / −6 across two files
  (`litellm/proxy/_experimental/mcp_server/mcp_server_manager.py` +37/−6;
  `tests/test_litellm/.../test_mcp_server_manager.py` +130/−0).

## What it changes

Two coordinated edits to the MCP OAuth discovery flow so an Azure Entra ID
issuer URL like `https://login.microsoftonline.com/<tenant>/v2.0` resolves
to working `authorization_url` and `token_url` instead of failing the
discovery probe:

1. **Removes the "must-challenge" guard at
   `mcp_server_manager.py:1488-1497`.** Pre-fix, when the MCP server
   `GET` returned 200 instead of a 401-with-WWW-Authenticate challenge,
   the discovery code logged `"MCP OAuth discovery unexpectedly succeeded
   for %s; server did not challenge"` and raised
   `RuntimeError("OAuth discovery must not succeed without a challenge")`.
   That branch is now replaced with a fallback that calls
   `await self._attempt_well_known_discovery(server_url)` to pull
   `authorization_servers` + `resource_scopes` from the resource's
   `.well-known/oauth-protected-resource`, then resolves the auth-server
   metadata and stamps `metadata.scopes = resource_scopes`.
2. **Adds two Azure-Entra-specific fallbacks in
   `_fetch_single_authorization_server_metadata`** at `:1681-1747`:
   - At `:1683-1687`, prepends
     `f"{issuer_url.rstrip('/')}/.well-known/openid-configuration"` to the
     candidate-URL list (the Entra v2.0 issuer's metadata lives at the
     issuer URL itself, not under the synthesized `base/.well-known/...`
     paths).
   - At `:1726-1747`, after all probes return None, falls back to
     `_build_azure_authorization_server_metadata(parsed)` which
     pattern-matches the issuer host (`login.microsoftonline.com`) plus
     a `<tenant>/v2.0` two-segment path and synthesizes
     `{base}/oauth2/v2.0/authorize` + `{base}/oauth2/v2.0/token`. Returns
     `None` for every other host so the fallback is bounded.

## Strengths

- **The "must-challenge" removal is correct.** RFC 8414 / draft MCP OAuth
  spec doesn't actually require the resource server to challenge before
  discovery — the hard requirement is on the *client* to attempt
  `.well-known/oauth-protected-resource` lookup. The previous
  `RuntimeError` was over-strict and broke any MCP server that exposes
  a `.well-known/oauth-protected-resource` document via 200 OK instead of
  via WWW-Authenticate hint. The new code at `:1491-1502` falls through
  to the well-known discovery flow regardless of probe status.
- **Issuer-as-metadata-URL fallback is the right Entra fix at the
  right level.** Entra v2.0's published OIDC discovery doc is at
  `https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration`
  (the issuer with `.well-known` appended), not at the synthesized
  `https://login.microsoftonline.com/.well-known/openid-configuration/{tenant}/v2.0`
  variant the previous candidate-list construction tried. Adding the
  literal `f"{issuer_url.rstrip('/')}/.well-known/openid-configuration"`
  to the candidate list at `:1683-1687` lets the existing probe loop
  succeed via the pre-existing GET-and-parse path rather than requiring
  any new fetch logic.
- **The `_build_azure_authorization_server_metadata` host-and-path guard
  is tightly bounded.** At `:1730-1737` the code rejects anything where
  `parsed_issuer_url.netloc != "login.microsoftonline.com"` OR the path
  isn't exactly two segments OR the second segment isn't `"v2.0"`. So
  the fallback only fires for the precise Entra v2.0 issuer shape and
  silently returns `None` for every other host (including
  `login.microsoftonline.us` GovCloud or `login.partner.microsoftonline.cn`
  China cloud — which need their own entries but won't get
  silently-wrong metadata).
- **Three new tests at `:731-861` pin the contract on all three branches:**
  (1) `test_descovery_metadata_probes_well_known_when_server_does_not_challenge`
  uses an `AsyncMock` 200 response and a patched well-known discovery to
  pin the new fall-through path; (2)
  `test_fetch_single_authorization_server_metadata_supports_azure_issuer_path`
  stubs the per-URL response so only `{issuer}/.well-known/openid-configuration`
  returns 200 and asserts the parsed Entra metadata round-trips; (3)
  `test_fetch_single_authorization_server_metadata_derives_azure_metadata`
  forces every candidate URL to 404 and asserts the
  `_build_azure_authorization_server_metadata` synthesis fallback fires.
  That's complete branch coverage for the three new code paths.
- **The fallback synthesis is purely deterministic** — no additional HTTP
  calls in the hot path when the synthesis fires. That's the right shape
  for a network-already-failed code path; the alternative (probe yet
  another candidate URL) would compound latency for a request that's
  already burned through the candidate list.

## Concerns / nits

- **The `_build_azure_authorization_server_metadata` synthesis returns
  metadata WITHOUT scopes** at `:1746-1749`. That's correct for the
  base case (the synthesis is purely URL construction and has no scope
  source), but the calling code at `:1493-1502` then does
  `if metadata is None and resource_scopes: return MCPOAuthMetadata(scopes=resource_scopes)`
  — i.e. only attaches `resource_scopes` if `metadata is None`, not if
  `metadata.scopes is None`. So the synthesis path silently drops the
  resource-scopes hint that the well-known discovery had already gathered.
  Worth a follow-up `if metadata is not None and metadata.scopes is None
  and resource_scopes: metadata.scopes = resource_scopes` — or, more
  simply, hoist the scope-stamping into a single block after the
  None-check. The PR's existing
  `if metadata is not None and resource_scopes: metadata.scopes = resource_scopes`
  at `:1500` does this when scopes are present, but the synthesis path
  starts with `metadata.scopes = None` and would be silently overwritten
  there too — so this is borderline. Worth tracing the exact failure case.
- **`login.microsoftonline.us` (Entra Gov)** and
  `login.partner.microsoftonline.cn` (Entra China) are not covered by the
  netloc guard. Customers in those clouds will fall through the synthesis
  and get `None` — i.e. discovery still fails, just less loudly than
  before. Either widening the guard to a `netloc.endswith(...)` set or
  documenting "Entra Gov/China customers must configure full
  authorization-server metadata explicitly" in the PR body would head off
  enterprise support requests.
- **The PR title says "azure entra discovery endpoint"** but the
  must-challenge guard removal is a much broader fix that helps every
  MCP server that returns 200 to the discovery probe (not just Azure).
  The two fixes are correctly bundled (they share the same `_descovery_metadata`
  function), but the PR body should call out the broader scope so
  reviewers reading the title aren't surprised by the second change.
- **Typo: `_descovery_metadata`** — the existing function name has
  "descovery" instead of "discovery" and the new tests inherit the
  misspelling (`test_descovery_metadata_probes_...`). Not introduced by
  this PR, but worth a follow-up rename in a separate cleanup commit
  before more API surface accumulates around the wrong name.
- **The `_attempt_well_known_discovery` and
  `_fetch_authorization_server_metadata` calls are sequenced** at
  `:1493-1502` so a failed `_attempt_well_known_discovery` (e.g. the
  resource doesn't expose `.well-known/oauth-protected-resource`)
  silently leaves `authorization_servers` empty and
  `_fetch_authorization_server_metadata` returns None. The function then
  reaches the bottom of the try block and... falls through to the
  HTTPStatusError handler? Worth tracing whether the empty-servers case
  returns `None` or raises — a single-line "no auth servers discovered"
  log at the would-be empty path would help operator debugging.
- **The Cargo of `mock_response.raise_for_status = MagicMock()`** at
  `:737-738` (not `AsyncMock`) is correct because the new code at
  `:1491` does `response.raise_for_status()` synchronously (no `await`),
  but if a future refactor wraps it in `await response.araise_for_status()`
  the test would silently still pass with the wrong method shape. A test
  comment naming the contract would help.

## Verdict

**merge-after-nits.** Two correct fixes (must-challenge removal +
Entra-specific synthesis) bundled in one PR with complete branch coverage
on the three new paths. The host-and-path guard for the synthesis
fallback is appropriately tight. Before merge: (1) trace the
`metadata.scopes is None` interaction with the resource-scopes hint to
confirm the synthesis path doesn't silently drop scopes, (2) widen the
netloc guard to cover `login.microsoftonline.us` /
`login.partner.microsoftonline.cn` or document the limitation, and (3)
update the PR title/body to reflect the broader must-challenge fix beyond
just Azure.
