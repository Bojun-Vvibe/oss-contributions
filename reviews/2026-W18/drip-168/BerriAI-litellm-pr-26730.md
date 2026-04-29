# BerriAI/litellm PR #26730 — fix: add optional TCP SO_KEEPALIVE support to aiohttp's TCPConnector

- Repo: BerriAI/litellm
- PR: https://github.com/BerriAI/litellm/pull/26730
- Head SHA: `b4fe3e4d9d1c13781a49c7649fae7de9d1031551`
- Author: yassinkortam
- Size: +230 / 0, 4 files

## Context

The reported failure mode: long provider responses (OpenAI/Azure stream
tail, slow Anthropic batch completion — anything ≥ 350s of idle TCP) get
killed mid-flight when litellm runs behind a NAT/LB whose idle timeout is
shorter than the provider response timeout. AWS NAT Gateway is the canonical
example (350s idle), and litellm's default httpx/aiohttp client opens TCP
sockets without `SO_KEEPALIVE`, so the kernel sends nothing during the long
wait, the NAT reaps the flow, and the eventual response arrives at a
forgotten 4-tuple. The fix is to wire `SO_KEEPALIVE` into aiohttp's
`TCPConnector` via the `socket_factory` kwarg (added in aiohttp 3.10) so the
kernel emits TCP probes that reset the NAT idle timer.

## What the diff actually does

Three production sites + one test file (~161 lines):

1. `litellm/constants.py:227-234` — adds four env-tunable knobs:
   `AIOHTTP_SO_KEEPALIVE` (bool, default `False`),
   `AIOHTTP_TCP_KEEPIDLE` (default 60s, idle-before-first-probe),
   `AIOHTTP_TCP_KEEPINTVL` (default 30s, between probes),
   `AIOHTTP_TCP_KEEPCNT` (default 5, probes-before-give-up). Default-off
   is the right call — opt-in semantics keep the NAT-Gateway fix from
   regressing latency-sensitive deployments where 60s idle probes are
   wasted bandwidth.

2. `litellm/llms/custom_httpx/http_handler.py:54-87` — adds
   `_AIOHTTP_SUPPORTS_SOCKET_FACTORY` via runtime
   `inspect.signature(TCPConnector.__init__).parameters` check (correct
   way to feature-detect the 3.10+ kwarg without pinning a version), and
   `_build_aiohttp_keepalive_socket_factory()` which returns either `None`
   (when disabled or aiohttp too old) or a closure that builds a
   `socket.socket(family, type, proto)`, sets `SO_KEEPALIVE = 1`, then
   conditionally sets `TCP_KEEPIDLE` / `TCP_KEEPALIVE` (Linux vs.
   Darwin: Linux uses `TCP_KEEPIDLE`, macOS uses `TCP_KEEPALIVE` for the
   same semantic), `TCP_KEEPINTVL`, `TCP_KEEPCNT` — each guarded by
   `hasattr(socket, ...)` so Windows (which has different/missing
   constants) doesn't crash. The factory is wired into both call sites at
   `:982-988` (per-request transport) and `litellm/proxy/proxy_server.py:672-693`
   (shared session for the proxy boot path).

3. Test file `tests/test_litellm/llms/custom_httpx/test_aiohttp_so_keepalive.py`
   exercises three branches: disabled (no `socket_factory` kwarg passed
   to `TCPConnector`), enabled-and-supported (factory is callable on the
   kwargs), enabled-but-old-aiohttp (no `socket_factory` kwarg, gracefully
   degraded). The `_invoke_connector_factory` helper drives the
   `_client_factory` lambda directly to force `TCPConnector` construction
   without depending on `_get_valid_client_session`'s internal branching
   — a clean isolation pattern.

## Risks and gaps

1. **The factory creates a new `socket.socket` rather than configuring an
   existing one**, which means the connection-setup code path skips
   anything aiohttp's default factory does. Diff shows `setblocking(False)`
   is set, which is what aiohttp needs. But aiohttp 3.10+'s default
   `socket_factory` may also call `set_inheritable(False)` (POSIX
   FD-leak hygiene under fork) and may set `IPV6_V6ONLY` on dual-stack
   sockets. If aiohttp's defaults shift in 3.11/3.12, this factory
   silently regresses against them. Cross-checking the upstream aiohttp
   `TCPConnector._make_socket` source for the same Python target the
   project supports would close this gap.

2. **No socket-level integration test**. Every test in the new file
   asserts on the kwargs *passed* to `TCPConnector` (mocked), not on the
   actual `SO_KEEPALIVE` value of a real socket. A `socket.fromfd` round-trip
   test (or `getsockopt(SOL_SOCKET, SO_KEEPALIVE)`) on the result of
   the factory invocation would pin the actual kernel state, which is
   what users actually need to know works. Three lines extra:
   ```
   sock = factory((socket.AF_INET, socket.SOCK_STREAM, 0))
   assert sock.getsockopt(socket.SOL_SOCKET, socket.SO_KEEPALIVE) == 1
   sock.close()
   ```

3. **`TCP_KEEPALIVE` on Darwin uses `IPPROTO_TCP` level**, which the diff
   gets right at line 80. But on macOS Big Sur+ the constant is named
   `TCP_KEEPALIVE` and the value is `0x10` (vs Linux `TCP_KEEPIDLE = 0x4`),
   semantically equivalent (idle-before-first-probe). The PR's
   `elif hasattr(socket, "TCP_KEEPALIVE")` branch is correct, but a
   one-line comment that this is the macOS equivalent (not a "fallback if
   the Linux name is missing") would prevent a future "simplify" pass
   from collapsing the branches incorrectly.

4. **Windows is silently a no-op for the per-knob settings** but
   `SO_KEEPALIVE` itself is supported on Windows and gets set. The probe
   timing is then governed by Windows' system-wide registry settings
   (`HKLM\SYSTEM\...\Tcpip\Parameters\KeepAliveTime`, default 7200s),
   which is much longer than the NAT idle timeout the fix is targeting.
   So on Windows, this PR sets `SO_KEEPALIVE` but doesn't actually fix
   the NAT-reap problem. A note in the env-var docstring or a
   `WARNING` log on Windows would prevent surprised deployments.

5. **Module-level `_AIOHTTP_SUPPORTS_SOCKET_FACTORY` is computed at import
   time**, which is fine for the global-aiohttp case but means a
   monkeypatched aiohttp (some test harnesses do this) won't be detected.
   The test at line 198 monkeypatches the constant directly, which works,
   but a `@functools.lru_cache(maxsize=1)` wrapper around the inspection
   would be more idiomatic and easier to override at runtime if needed.

6. **No test for the `proxy_server.py` shared-session call site**. The
   diff wires the factory into `_initialize_shared_aiohttp_session` at
   `proxy_server.py:686-693`, but the test file only exercises the
   per-request transport path. Asserting the same kwarg presence on the
   shared-session connector would catch a future refactor that drops the
   wiring on one path.

## Suggestions

- Add a real-socket integration test asserting
  `sock.getsockopt(SOL_SOCKET, SO_KEEPALIVE) == 1` on the factory
  output, so the test verifies kernel state not just kwargs.
- Comment that the `TCP_KEEPALIVE` branch is the macOS equivalent of
  `TCP_KEEPIDLE`, not a fallback — prevents wrong "simplify" passes.
- Add a `WARNING` log (or env-var docstring) noting that on Windows the
  per-knob timing isn't honored and probe timing falls back to the OS
  registry default of 7200s.
- Mirror the per-request-transport test path with a shared-session test
  for `_initialize_shared_aiohttp_session`'s wiring.
- Optionally compare the factory's setup to upstream aiohttp's
  `TCPConnector._make_socket` to confirm it's not skipping any
  default-hygiene calls (`set_inheritable`, `IPV6_V6ONLY` on dual-stack).

## Verdict

**merge-after-nits** — opt-in default is correct, runtime aiohttp
feature-detection is the right shape (vs version pinning), the
Linux/Darwin `TCP_KEEPIDLE`/`TCP_KEEPALIVE` distinction is handled
correctly. Nits are about real-socket vs mocked-kwargs test depth, the
Windows behavioral footnote, and tightening parity with aiohttp's own
default factory.
