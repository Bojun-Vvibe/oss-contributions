# BerriAI/litellm #26841 — chore(mcp): require trusted-proxy gate before honouring X-Forwarded-* on OAuth discovery

- **URL:** https://github.com/BerriAI/litellm/pull/26841
- **Head SHA:** `c3fcafb1bf9f8da086eb7d540360eded9dbeb39b`
- **Files:** `litellm/proxy/_experimental/mcp_server/discoverable_endpoints.py` (+8/-15), `litellm/proxy/auth/ip_address_utils.py` (+59/-0), `tests/test_litellm/proxy/_experimental/mcp_server/test_discoverable_endpoints.py` (+~159 / fixture wiring)
- **Verdict:** `merge-as-is`

## What changed

Closes a real OAuth-discovery URL-poisoning vuln in the MCP discoverable endpoints (`/oauth/authorize`, `/oauth/token`, `/.well-known/oauth-protected-resource`, `/.well-known/oauth-authorization-server`, `/oauth/register`).

The previous `get_request_base_url(request)` at `discoverable_endpoints.py:33-79` *unconditionally* honored `X-Forwarded-Proto` / `X-Forwarded-Host` / `X-Forwarded-Port` headers when constructing the base URL embedded in OAuth metadata responses. Any unauthenticated caller could send an HTTP request with `X-Forwarded-Host: attacker.example.com`, `X-Forwarded-Proto: https`, and the discovery response would advertise `https://attacker.example.com/...` as the issuer / `redirect_uri` base — well-known OAuth open-redirect / mix-up attack class. With MCP's `redirect_uri` being the place where the OAuth code is delivered, an attacker who can control the issuer URL in discovery can steer authorization codes to themselves.

Fix is two-piece:

1. **New `IPAddressUtils.is_request_from_trusted_proxy(request, general_settings=None)`** at `ip_address_utils.py:113-167`. Returns `True` iff *both* `general_settings["use_x_forwarded_for"] is True` *and* `general_settings["mcp_trusted_proxy_ranges"]` is configured-and-the-direct-IP-matches. Direct IP comes from `request.client.host` at `:166`, then `IPAddressUtils.is_trusted_proxy(direct_ip, parse_trusted_proxy_networks(trusted_ranges))`.

2. **Discovery endpoint gate** at `discoverable_endpoints.py:52-53`:
   ```python
   if not IPAddressUtils.is_request_from_trusted_proxy(request):
       return base_url
   ```
   placed *before* the X-Forwarded-* parse, so untrusted callers get the literal `request.base_url` (which FastAPI sources from the actual TCP `Host:` header / scheme — controlled by the proxy operator, not the caller).

3. **Operator-help one-shot warning** at `:151-163`. When `use_x_forwarded_for=True` is set but `mcp_trusted_proxy_ranges` is not, the *first* request through the gate logs a multi-line `verbose_proxy_logger.warning` explaining the headers will not be trusted and how to opt in. Module-level `_warned_xff_without_trusted_ranges` flag at `:16-18` ensures one-shot — won't spam logs at request rate.

4. **Test wiring** uses a new `trust_xff` fixture at `test_discoverable_endpoints.py:26-39` that monkey-patches `IPAddressUtils.is_request_from_trusted_proxy` to `True` for the existing X-Forwarded-Proto regression tests. Five existing tests (`test_authorize_endpoint_respects_x_forwarded_proto`, `_token_endpoint_`, `_oauth_protected_resource_`, `_oauth_authorization_server_`, `_register_client_respects_x_forwarded_proto`) get `@pytest.mark.usefixtures("trust_xff")`. The trust gate's own behavior is referenced as covered separately by `test_get_request_base_url_xff_trust_gate` (named in the fixture docstring at `:33`).

## Why it's right

- **Threat model is real.** OAuth metadata endpoints are unauthenticated by spec — they have to be, since clients consume them before any auth handshake. The previous `get_request_base_url` unconditionally trusting forwarded headers meant any attacker with HTTP access could steer the advertised authorization-server URL. With MCP OAuth being a relatively new code path, this is the kind of thing that gets shipped permissively-on-purpose for ease-of-deploy and then has to be tightened. This PR is the tightening.
- **Two-condition gate, not one, is correct.** Just `use_x_forwarded_for=True` isn't sufficient — that flag is widely set on litellm proxies behind nginx without the operator necessarily understanding it newly trusts attacker-controlled headers on this specific endpoint family. Requiring `mcp_trusted_proxy_ranges` *as well* forces the operator to opt in with a CIDR list, and the warning at `:151-163` makes the upgrade footprint visible (operator who upgrades and doesn't add the CIDR sees one warning, OAuth still works but uses literal base URL — degrades gracefully toward safe).
- **Direct-IP check via `request.client.host` is the right source.** The first-hop TCP source is the only IP value that *isn't* attacker-controlled. Any `X-Forwarded-For` parsing here would be circular (the very flag being gated). The `is_trusted_proxy(direct_ip, networks)` helper is reused from existing internal-IP logic at `:55-107`, so the CIDR-parsing surface is shared.
- **One-shot warning, not per-request.** `_warned_xff_without_trusted_ranges` module-global at `:18` flips the first time the partial-config path triggers and stays flipped. Operators who upgrade and forget to set the CIDR see a single, actionable message rather than log spam. The message names the exact setting (`mcp_trusted_proxy_ranges`) and the file location semantic (`general_settings`) — directly actionable.
- **Test migration is the right shape.** Five existing tests asserted the X-Forwarded-Proto-honoring *behavior* — those tests still need to pass to prove the new code didn't regress *the trusted path*. Wrapping them with `usefixtures("trust_xff")` keeps the behavior assertion intact while making the trust-gate explicit. Adding a separate `test_get_request_base_url_xff_trust_gate` (referenced in fixture docstring at `:33`) for the gate itself separates concerns: gate tests own gate behavior, header-parsing tests own header parsing. Healthy decomposition.
- **Fail-closed default** is the choice that matters. When `is_request_from_trusted_proxy` returns `False`, the function early-returns `base_url` unchanged — falling through to the *literal* request URL, which is whatever the actual TCP listener saw. That's safe by construction: an attacker can't make `request.base_url` lie because FastAPI sources scheme from the ASGI `scope["scheme"]` (set by the server based on actual TLS termination) and host from the first-hop `Host:` header (set by the actual TCP client, which is the proxy if behind one).

## Worth-noting (non-blocking)

- The lazy-import-on-default-path at `:142-149` (`from litellm.proxy.proxy_server import general_settings as proxy_general_settings`) avoids a circular import between `ip_address_utils` and `proxy_server`. Reasonable, but it does mean unit tests that pass `general_settings=None` will trigger an import at first call. The `try/except ImportError → general_settings = {}` fallback at `:148-149` keeps tests working in isolation. Worth a comment naming the circular-import reason at the import.
- `request.client.host if request.client else None` at `:165`. `request.client` is `None` for some test/ASGI shapes; falling through to `None` and then `is_trusted_proxy(None, networks)` presumably returns `False` (worth verifying). The code is correct; a one-line `# request.client is None for in-memory ASGI test shapes` comment would help future maintainers.
- The five existing X-Forwarded-Proto regression tests now require the `trust_xff` fixture. Anyone adding a *new* X-Forwarded-* test in this file in the future may forget the fixture and write a test that asserts the *opposite* of the trust gate (i.e. asserts X-Forwarded-Proto is honored without setting up the trust). The fixture docstring at `:33` calls out the convention; a brief comment at the top of the test file restating it would be belt-and-braces.
- The PR doesn't touch the *non*-MCP OAuth endpoints (e.g. the management UI's own SSO flow). That's correct — those have their own trust assumptions and gating those is out-of-scope. But this surface-by-surface tightening pattern probably wants a tracking issue to ensure the rest follow.

## Risk

Low for correctly-configured deployments (operator sets `mcp_trusted_proxy_ranges` to their LB CIDR, behavior is unchanged). Medium for *misconfigured* deployments where the operator has `use_x_forwarded_for=True` and a real proxy in front but no `mcp_trusted_proxy_ranges` set — those will start advertising the proxy's literal `Host:` value (rather than the operator-intended one) in OAuth discovery responses. The mitigation is the warning at `:151-163`, plus this is exactly the security-hardening tradeoff the operator is opting into. PR notes should call this out explicitly in the changelog as a behavior change requiring `mcp_trusted_proxy_ranges` configuration on upgrade.
