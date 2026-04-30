# BerriAI/litellm PR #26849 — chore(mcp): SSRF guard on OAuth metadata discovery follow-up fetches

- PR: https://github.com/BerriAI/litellm/pull/26849
- Head SHA: `1bb04cdc72dc53a25b42a0b71259b059eea65748`
- Files touched: `litellm/proxy/_experimental/mcp_server/mcp_server_manager.py` (+104/-10), `tests/test_litellm/proxy/_experimental/mcp_server/test_mcp_server_manager.py` (+245/-6)

## Specific citations

- New `_is_safe_metadata_url(url, server_url)` static at `mcp_server_manager.py:1504-1567` is the load-bearing security primitive. Decision tree:
  1. Reject anything not `http`/`https` or with no hostname (`:1535-1536`).
  2. Compute target/base ports defaulting on scheme (`:1538-1539`).
  3. **Same-authority allow** (`:1540-1547`): scheme + lowercased hostname + port match → allow without DNS. This covers RFC 9728 §3.3 PRM published at the resource server itself plus admin-constructed well-known endpoints.
  4. **DNS resolution + per-IP block** (`:1549-1565`): `socket.getaddrinfo(host, port, type=SOCK_STREAM)` → for each `(family, type, proto, canonname, sockaddr)` extract `sockaddr[0]` and check via `_is_blocked_ip` (the proxy-wide outbound blocklist).
  5. Empty resolution or `gaierror` → reject.
- The doc comment at `:1559-1562` explicitly names the **DNS rebinding window** between this resolution and httpx's request-time resolution as a known limitation, with the mitigation being the same-authority pin (an attacker who controls the resource server is already in scope).
- Three call sites threaded with `server_url`:
  - `_fetch_oauth_metadata_from_resource(resource_metadata_url, server_url)` at `:1686` — was single-arg.
  - `_fetch_authorization_server_metadata(authorization_servers, server_url)` at `:1771` — was single-arg.
  - `_fetch_single_authorization_server_metadata(issuer_url, server_url)` at `:1779` — was single-arg.
- Three guard-applies:
  - `:1693-1699` PRM URL guard with `verbose_logger.warning(...)` and `return [], None`.
  - `:1804-1811` AS-metadata candidate-URL guard with `continue` (skips one candidate, tries the next).
  - The pre-existing well-known discovery loop already constructs same-authority URLs and goes through `_fetch_oauth_metadata_from_resource` so the guard fires there too.
- **Critical hardening**: both fetch sites now pass `follow_redirects=False` to `httpx.client.get` at `:1707` and `:1820`, with comments at `:1704-1706` and `:1817-1819` explicitly naming "redirects bypass the SSRF guard since the new `Location` is not re-checked against `server_url`".
- Test class `TestOAuthDiscoverySSRFGuard` at `test_mcp_server_manager.py:2959+` covers:
  - `_patch_resolves` helper mocks `socket.getaddrinfo` deterministically (`:2965-2986`).
  - Same-authority allow without DNS at `:2988-2995`.
  - Same-host different-port falls to DNS check, blocked on private IP at `:2997-3007`.
  - **Comprehensive parametrized blocklist test** at `:3009-3024`: `127.0.0.1`, `10.0.0.5`, `172.16.0.1`, `192.168.1.1`, **`169.254.169.254`** (cloud IMDS — AWS/Azure/GCP), **`100.100.100.200`** (Alibaba Cloud metadata), `0.0.0.0`, `::1`, `fe80::1`, `fc00::1`. All asserted blocked.

## Verdict: merge-as-is

## Concerns / nits

1. The IMDS-coverage test at `:3018` includes both AWS-style (`169.254.169.254`) and Alibaba (`100.100.100.200`) but not GCP's *additional* `metadata.google.internal` hostname (which resolves to `169.254.169.254` so it's covered transitively, but the comment at `:3018` could mention it). Negligible; the IP coverage is what matters and `_is_blocked_ip` is the source of truth.
2. **DNS rebinding ack** at `:1559-1562` is the right level of honesty. Worth filing a follow-up issue tracking either (a) connecting via a resolver that pins the resolved IP into the httpx connection, or (b) using a custom transport that re-checks the connected peer address. The current shape is appropriate for the threat model named ("an attacker who also controls the resource server is already in scope").
3. **`follow_redirects=False`** comments at `:1704-1706` and `:1817-1819` are *exemplary* — they name *why* the redirect-disable is the right choice (Location target wouldn't be re-checked) rather than "we don't follow redirects because spec". Future contributors won't accidentally re-enable it.
4. Test signature change — `_fetch_single_authorization_server_metadata(issuer, issuer)` at the pre-existing tests `:815-816, 856-858` uses the issuer URL as both arg slots; this is a pragmatic test-only same-authority shortcut, but the comment at `:809-811` names it ("The Azure issuer is cross-origin against the server_url — use the issuer itself as server_url so the test exercises the well-known fetch logic without needing real DNS"). Good.
5. The `_is_blocked_ip` import at `:46` reuses the proxy-wide outbound blocklist — single source of truth for "what's a private IP" across the proxy. If a future security advisory adds a new IPv6 ULA or cloud metadata range, both this guard and the proxy-wide outbound check pick it up automatically. Right shape.
6. The `same_authority` pin is computed *before* DNS at `:1540-1547` so a same-host same-port URL never hits getaddrinfo — both a perf optimization and a DNS-rebinding-tightening (the legitimate "we trust our own server_url" pin doesn't depend on resolution).
