# Review: BerriAI/litellm#26730 — fix: added keepalive args for aiohttp tcpconnector

- **Author**: yassinkortam
- **Head SHA**: `0e1ba362ebf021f44f75d579937737f98ba64c9c`
- **Diff**: +182 / −0 across 4 files
- **Linear**: LIT-2592

## What it does

Adds opt-in TCP `SO_KEEPALIVE` (with platform-aware `TCP_KEEPIDLE` / `TCP_KEEPINTVL` / `TCP_KEEPCNT` tuning) to the aiohttp `TCPConnector` used by both the in-process httpx-on-aiohttp transport and the proxy's shared aiohttp session. The motivating failure is the well-known mismatch between AWS NAT Gateway's 350-second idle timeout and OpenAI/Azure's up-to-600-second slow-response window: the kernel sends nothing on the wire while the provider thinks; the NAT reaps the flow; the response never arrives.

The implementation:

- **`litellm/constants.py:227-235`** adds four env-driven constants: `AIOHTTP_SO_KEEPALIVE` (default `False`, opt-in), `AIOHTTP_TCP_KEEPIDLE` (60s), `AIOHTTP_TCP_KEEPINTVL` (30s), `AIOHTTP_TCP_KEEPCNT` (5). Defaults are conservative (60+30*5=210s before connection drop) and well below the 350s NAT idle.
- **`litellm/llms/custom_httpx/http_handler.py:63-100`** introduces a one-time module-level capability probe `_AIOHTTP_SUPPORTS_SOCKET_FACTORY = "socket_factory" in inspect.signature(TCPConnector.__init__).parameters` (aiohttp ≥3.10 only) and a `_build_aiohttp_keepalive_socket_factory()` helper that returns either `None` (when disabled or unsupported) or a callable matching aiohttp's `socket_factory` contract: takes `addr_info: Tuple[Any, ...]`, returns a non-blocking `socket.socket(family, type, proto)` with `SO_KEEPALIVE=1` plus the three platform-conditional TCP knobs (`TCP_KEEPIDLE` on Linux, `TCP_KEEPALIVE` on Darwin/macOS as documented in the inline comment, `TCP_KEEPINTVL`/`TCP_KEEPCNT` where available).
- **`http_handler.py:982-987`** in `_create_aiohttp_transport` and **`proxy_server.py:672-691`** in `_initialize_shared_aiohttp_session` both consult `_build_aiohttp_keepalive_socket_factory()` and only attach `socket_factory` to `transport_connector_kwargs` / `connector_kwargs` when the helper returns non-None. Clean opt-in with no behavior change for users who don't set `AIOHTTP_SO_KEEPALIVE=true`.
- **`tests/test_litellm/llms/custom_httpx/test_aiohttp_so_keepalive.py`** (+115) is a four-test surface using `monkeypatch` to flip the constants and `unittest.mock.patch` to spy on `TCPConnector` kwargs:
  - `test_socket_factory_omitted_when_disabled` — flag off → kwarg absent
  - `test_socket_factory_attached_when_enabled` — flag on, supports kwarg → kwarg present and callable
  - `test_socket_factory_skipped_on_old_aiohttp` — flag on, doesn't support kwarg → kwarg absent (no crash)
  - `test_socket_factory_sets_keepalive_options` — exercises the factory itself with a `MagicMock(spec=socket.socket)`, asserts `setblocking(False)` and the exact `setsockopt` triple-tuples, with `if hasattr(socket, "TCP_KEEPIDLE") ... elif hasattr(socket, "TCP_KEEPALIVE")` branches matching the implementation's platform fork.

## Concerns

1. **PR's pre-submission checklist is fully unchecked** — no Greptile review, no CI links, no claim that `make test-unit` passes. Given the change touches both transport paths *and* the proxy's shared session bootstrap, both should have green CI before merge. Test surface in-PR is decent but doesn't cover the actual integration (no aiohttp server stood up).

2. **`socket.setblocking(False)` is set on the bare socket before aiohttp gets it.** aiohttp normally creates non-blocking sockets internally; setting it ourselves is fine for standard `AF_INET`/`AF_INET6` SOCK_STREAM, but worth confirming aiohttp's `socket_factory` contract doesn't expect it to be left blocking and let the loop handle it. The aiohttp 3.10 release notes for `socket_factory` should clarify; the PR doesn't cite them. If aiohttp expects blocking, this is fine on Linux/Darwin but could subtly break on Windows (which doesn't use this code path heavily, but still).

3. **`AIOHTTP_TCP_KEEPIDLE=60` defaults to one probe per minute, which is aggressive.** If a deployment has tens of thousands of long-lived connections (provider-side keepalive pools), that's tens of thousands of unnecessary keepalive probes per minute. The `TCPConnector(keepalive_timeout=AIOHTTP_KEEPALIVE_TIMEOUT=120)` default already reaps idle conns at 2 minutes, so KEEPIDLE=60 fires *during* the still-warm window. A default of `300` (closer to NAT idle / 2) would be less probe-spammy and still well under the 350s NAT timeout. Document in the constants block what the math is for the recommended values and what the user's tradeoff is when overriding.

4. **Two call sites that build aiohttp connectors duplicate the `if socket_factory is not None: kwargs["socket_factory"] = socket_factory` pattern.** Worth hoisting both kwargs-builders into `_build_aiohttp_connector_kwargs(base_kwargs: Dict)` in `http_handler.py` so the next person adding a connector option doesn't have to remember to update both.

5. **`socket.SO_KEEPALIVE=1` reset behavior on connection reuse.** aiohttp's connector reuses sockets across requests. The factory only fires on socket *creation*, so reused sockets keep their keepalive — good. But sockets in the pool that were created *before* the user flipped the env var won't have it. This is fine for typical deployment (env var set at boot), but it should be documented that the flag is read once at module import via `AIOHTTP_SO_KEEPALIVE = os.getenv(...)` and runtime toggles via dynamic config won't take effect.

6. **No test for the proxy `_initialize_shared_aiohttp_session` path.** The four tests cover `_create_aiohttp_transport` only. A fifth test patching the proxy bootstrap function and asserting `connector_kwargs["socket_factory"]` is set under the same conditions would close the symmetry.

## Verdict

**merge-after-nits** — correct opt-in fix for a real production failure mode (NAT-gateway-vs-LLM-latency mismatch), with an `inspect.signature`-based capability probe that gracefully degrades on old aiohttp. Land after the PR template is filled in (CI links, unit tests passing), the `_initialize_shared_aiohttp_session` path picks up the same test coverage, and the default `AIOHTTP_TCP_KEEPIDLE=60` is reconsidered or its tradeoff documented in the constants block.

