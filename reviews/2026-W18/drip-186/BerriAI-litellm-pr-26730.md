---
pr: BerriAI/litellm#26730
sha: 848b79acb5e8f0d36e3b16321dc0fdabfaecd581
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# fix: add optional TCP SO_KEEPALIVE support to aiohttp's TCPConnector

URL: https://github.com/BerriAI/litellm/pull/26730
Files: `litellm/constants.py`, `litellm/llms/custom_httpx/http_handler.py`,
`litellm/proxy/proxy_server.py`,
`tests/test_litellm/llms/custom_httpx/test_aiohttp_so_keepalive.py`
Diff: 240+/0-

## Context

Real bug class with operator pain: aiohttp's default-built TCP sockets
ship without `SO_KEEPALIVE`, so during long provider calls (OpenAI /
Azure can run up to 600s for a single completion) the kernel emits
nothing on the wire, and any NAT or LB on the path with an idle timeout
shorter than the response budget reaps the flow before the response
arrives. AWS NAT Gateway's 350s idle timeout is the canonical case
called out in the docstring at `http_handler.py:78-82`. Without
`SO_KEEPALIVE`, the symptom on the proxy side is `ClientConnectionError`
/ `ServerDisconnectedError` after exactly the NAT idle timeout, with no
visible cause in either logs or metrics.

The fix is opt-in env-gated `SO_KEEPALIVE` plus the three Linux-side
companion knobs `TCP_KEEPIDLE` / `TCP_KEEPINTVL` / `TCP_KEEPCNT`,
plumbed through aiohttp's `socket_factory` kwarg (added in 3.10) at
both shared-session paths (`http_handler.py:_create_aiohttp_transport`
and `proxy_server.py:_initialize_shared_aiohttp_session`).

## What's good

- Four constants at `constants.py:227-237` with sane defaults
  (`KEEPIDLE=60`, `KEEPINTVL=30`, `KEEPCNT=5` — first probe at 60s,
  retry every 30s for 5 attempts → max 210s before TCP gives up,
  comfortably under both AWS NAT's 350s and the 600s response budget)
  and the load-bearing comment at `:227-232` naming the exact NAT
  Gateway timeout that motivated the change. Future ops engineers
  reading this will not have to git-blame to understand why these
  numbers exist.
- Version detection at `http_handler.py:65-67` via
  `inspect.signature(TCPConnector.__init__).parameters` is the right
  approach for "added in aiohttp 3.10" — a try/except on actually
  passing the kwarg would also work but the introspection is cleaner
  and runs once at import time.
- Cross-platform handling at `:104-117` for the macOS vs Linux divide:
  Darwin's `TCP_KEEPALIVE` does what Linux's `TCP_KEEPIDLE` does
  (idle-before-first-probe), and the `hasattr`/`elif hasattr` chain
  picks the right symbol without preprocessor games.
- The factory itself returns `None` cleanly when either the env flag
  is off or aiohttp is too old (`:88-89`), so all the call-site
  plumbing degrades to "don't pass `socket_factory` kwarg at all"
  instead of "pass it as None and hope the version accepts that".
- 161-line dedicated test file `test_aiohttp_so_keepalive.py` covers
  the matrix (env on/off, aiohttp old/new, Linux/Darwin symbol
  selection, factory returns valid socket).
- `sock.setblocking(False)` at `:97` is necessary — aiohttp expects
  non-blocking sockets and forgetting this would deadlock the event
  loop on first write.

## Nits / follow-ups

- The duplicate factory invocation at `http_handler.py:996-1000` and
  `proxy_server.py:687-690` is correct but invites drift. Lift the
  six-line `socket_factory = _build_aiohttp_keepalive_socket_factory();
  if socket_factory is not None: kwargs["socket_factory"] = socket_factory`
  block into a small `apply_keepalive_socket_factory(kwargs)` helper
  so the next caller can't forget half of it.
- `_AIOHTTP_SUPPORTS_SOCKET_FACTORY` at `:65` is computed at module
  import — fine for the common case, but the lazy `_build_aiohttp_*`
  call at request-build time means a downstream that monkey-patches
  `TCPConnector` later won't be re-detected. Worth a one-line comment
  documenting the import-time-detection contract.
- No Windows code path comment. Windows aiohttp also supports
  `socket_factory`, and the `hasattr(socket, "TCP_KEEPIDLE")` check
  will return False, so on Windows the user gets `SO_KEEPALIVE` but
  none of the three timing knobs — which means the kernel uses
  Windows' registry-controlled defaults (typically 2-hour KeepAliveTime).
  Worth either a docstring note or a Windows code path that uses
  `SIO_KEEPALIVE_VALS` via `sock.ioctl()`.
- The `factory(addr_info)` signature at `:91` unpacks
  `addr_info[0..2]` positionally — fragile against aiohttp tightening
  the contract to a typed tuple later. A `family, type_, proto, *_ =
  addr_info` would be more defensive.
- The "350s NAT idle vs 600s OpenAI response" framing in the docstring
  is great context but also an argument for `KEEPIDLE=60` being
  conservative — at 60s probe-start with 30s intervals, the NAT sees
  TCP traffic every 60s+30s once an idle period kicks in, which is
  ~6× more frequent than strictly needed to defeat the 350s timer.
  Worth either documenting that the conservative defaults trade a
  slight per-connection overhead for safety against shorter-timeout
  intermediaries (some corporate proxies are 60-90s), or letting the
  defaults be more relaxed.
- No integration test that actually opens a socket and asserts the
  options stuck via `getsockopt` — the unit test at the new file
  presumably mocks `socket.socket`. A single integration test that
  binds to localhost and reads back `SO_KEEPALIVE` / `TCP_KEEPIDLE`
  would fence the "we set it but the kernel ignored it" failure mode.

## Verdict

`merge-after-nits` — solves a real, well-understood operator pain with
the right opt-in posture, sound defaults, and good cross-platform
handling. The duplicate plumbing at two call sites and the missing
Windows timing-knob path are the two items worth addressing in a
follow-up; nothing here blocks shipping the env-gated fix.
