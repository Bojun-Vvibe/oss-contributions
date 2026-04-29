# BerriAI/litellm #26730 — opt-in TCP SO_KEEPALIVE on aiohttp connector

- **PR:** https://github.com/BerriAI/litellm/pull/26730
- **Title:** `fix: add optional TCP SO_KEEPALIVE support to aiohttp's TCPConnector`
- **Author:** yassinkortam
- **Head SHA:** b4fe3e4d9d1c13781a49c7649fae7de9d1031551
- **Files changed:** 4 (`litellm/constants.py`, `litellm/llms/custom_httpx/http_handler.py`,
  `litellm/proxy/proxy_server.py`, new
  `tests/test_litellm/llms/custom_httpx/test_aiohttp_so_keepalive.py`),
  +230 / −0
- **Verdict:** `merge-after-nits`

## What it does

Adds opt-in `SO_KEEPALIVE` (and platform-appropriate
`TCP_KEEPIDLE`/`TCP_KEEPINTVL`/`TCP_KEEPCNT`) on aiohttp TCP sockets so
long-running provider calls don't get reaped by NAT/LB idle timers. The
canonical case the PR body cites: AWS NAT Gateway has a 350s idle timer,
OpenAI/Azure responses can take up to ~600s; without keepalive probes
the kernel sends nothing during the wait and the NAT silently kills the
flow.

Mechanics:

1. New env-tunable constants in `constants.py:227-235`:
   `AIOHTTP_SO_KEEPALIVE` (default `False`), `AIOHTTP_TCP_KEEPIDLE=60`,
   `AIOHTTP_TCP_KEEPINTVL=30`, `AIOHTTP_TCP_KEEPCNT=5`.
2. `_build_aiohttp_keepalive_socket_factory()` in
   `http_handler.py:67-99` returns a `socket_factory` callable suitable
   for `aiohttp.TCPConnector(socket_factory=...)` (aiohttp 3.10+).
   Returns `None` when the feature is disabled or aiohttp is too old —
   detected once via
   `_AIOHTTP_SUPPORTS_SOCKET_FACTORY = "socket_factory" in inspect.signature(TCPConnector.__init__).parameters`.
3. Wired into both `_create_aiohttp_transport` (line ~982-986) and the
   shared session in `proxy_server.py:686-689`.
4. Two unit tests assert: (a) `socket_factory` is **omitted** from
   `TCPConnector` kwargs when `AIOHTTP_SO_KEEPALIVE=False`; (b) it is
   **attached** when enabled.

## What's good

- Opt-in default. The right call: NAT-induced timeouts are a fairly
  specific deployment shape, and forcing keepalive globally would
  surprise the long tail of small-scale users. `AIOHTTP_SO_KEEPALIVE`
  defaults to `False`.
- Cross-platform handling done with `hasattr` probes rather than
  `sys.platform` matching:

  ```python
  if hasattr(socket, "TCP_KEEPIDLE"):
      sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPIDLE, ...)
  elif hasattr(socket, "TCP_KEEPALIVE"):
      sock.setsockopt(socket.IPPROTO_TCP, socket.TCP_KEEPALIVE, ...)
  ```

  This is the right pattern — Linux exposes `TCP_KEEPIDLE`, macOS uses
  `TCP_KEEPALIVE` for the same semantic, Windows exposes neither
  through Python's stdlib so we silently no-op (which is fine because
  Windows manages keepalive timing globally).
- Aiohttp version detection lives at module load time
  (`_AIOHTTP_SUPPORTS_SOCKET_FACTORY`), not per-request. No perf hit on
  the hot path.
- Comments explain *why*, not what — the constants block at
  `constants.py:227-233` and the docstring on
  `_build_aiohttp_keepalive_socket_factory` both describe the NAT
  reaping problem and cite exact AWS-vs-provider numbers, which is the
  kind of context that survives a long blame chain.
- Default tunings (60s idle, 30s interval, 5 probes → first probe at
  60s, declared dead at ~210s) are well below AWS NAT's 350s and
  comfortably cover the OpenAI/Azure window.

## Nits / risks

- `_invoke_connector_factory` in the test file at lines ~5-16 reaches
  into `transport._client_factory()` directly — a private attribute.
  This is brittle if `LiteLLMAiohttpTransport` is ever refactored to
  hide that lambda. A comment already justifies the choice ("avoids
  relying on `_get_valid_client_session`'s internal branching"), so
  not blocking, but consider exposing a small test seam like
  `transport._build_session_for_test()` so the unit test isn't coupled
  to private state.
- `setblocking(False)` is set in the factory (line ~83). aiohttp itself
  also calls `setblocking(False)` on sockets it gets from
  `socket_factory`, so this is harmless but redundant — and worth a
  one-line comment noting the redundancy is intentional defensive code.
- `socket_factory` is called for every new connection (no pooling at
  this layer). For a high-QPS proxy this means ~4 extra `setsockopt`
  syscalls per new connection — negligible, but worth noting in the
  PR body for capacity planners.
- The two unit tests verify wiring, but there's no integration test
  that an actual long-idle TCP flow stays alive. That's hard to
  exercise in CI; consider opening a follow-up issue with a manual
  reproduction recipe (curl + `iptables` packet drop or `tc netem`
  rules) so the QA path is documented.

## Verdict rationale

`merge-after-nits`: solid feature, well-comented and opt-in by default,
correctly cross-platform. The nits are about test-seam ergonomics and
documentation — none block the production correctness of the code.
