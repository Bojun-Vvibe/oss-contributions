---
pr: 4722
repo: browser-use/browser-use
sha: 4fa600a38783d92213c8e98ec12785d75a7d12af
verdict: merge-after-nits
date: 2026-04-25
---

# browser-use/browser-use#4722 — Guard direct CDP helpers during reconnect

- **URL**: https://github.com/browser-use/browser-use/pull/4722
- **Author**: euyua9

## Summary

Adds a small guard, `_ensure_cdp_ready_for_command`, that direct CDP
storage/cookie helpers must `await` before issuing `Storage.getCookies`
/ `Storage.setCookies` / `Storage.clearCookies` /
`export_storage_state`. The guard waits for an in-flight reconnect via
`self._reconnect_event` (bounded by `RECONNECT_WAIT_TIMEOUT`) and
raises `ConnectionError` with a context-specific message if either the
wait times out or CDP is unavailable. Three new regression tests
exercise success, timeout, and `export_storage_state` flows.

## Reviewable points

- **Guard placement** (`browser_use/browser/session.py:1357-1373`):

  ```python
  async def _ensure_cdp_ready_for_command(self, operation_name: str) -> None:
      if self.is_cdp_connected:
          return
      if self.is_reconnecting:
          wait_timeout = self.RECONNECT_WAIT_TIMEOUT
          ...
          await asyncio.wait_for(self._reconnect_event.wait(), timeout=wait_timeout)
          if self.is_cdp_connected:
              return
          raise ConnectionError(f'{operation_name}: reconnection finished but CDP is still not connected')
      raise ConnectionError(f'{operation_name}: CDP is not connected')
  ```

  Three branches, each terminal. Good readability. One race window:
  between the `is_cdp_connected` check and the `is_reconnecting`
  check, a reconnect could *start*. That's not a real bug because
  the next call to `_ensure_cdp_ready_for_command` would see
  `is_reconnecting=True` and wait — but worth confirming
  `_reconnect_event` is not a one-shot that already-fired prior to
  this call, otherwise `wait()` returns immediately and the second
  `is_cdp_connected` check correctly raises. The test
  `test_cdp_command_waits_for_reconnect_success` uses
  `connected.side_effect = [False, True]` which proves the
  re-checked-after-wait contract; `_reconnect_event.set()` is called
  from a background task in the test, so the path *is* exercised.

- **Caller integration** (lines 1380-1385, 3328-3370):
  `export_storage_state`, `_cdp_get_cookies`, `_cdp_set_cookies`,
  `_cdp_clear_cookies` all gain a one-line `await
  self._ensure_cdp_ready_for_command('<name>')` prefix. Coverage
  question: are there other direct-CDP-helper sites that *also* need
  this guard but didn't get patched? A quick grep for
  `cdp_session.cdp_client.send.` in `session.py` would tell us. From
  the diff alone, only the storage/cookie helpers are wired — if
  there's a `_cdp_get_local_storage` or similar, that would be a
  follow-up. Worth the maintainer eyeballing this list.

- **Error semantics**: `ConnectionError` is the right base class
  (consistent with how callers higher up handle CDP loss). The
  context-specific messages (`'export_storage_state:
  reconnection wait timed out after 0.2s'`) make logs greppable.

- **Test quality**: the new tests at the bottom of
  `tests/ci/browser/test_session_start.py` cover three real branches:
  (1) reconnect-then-success (using a `PropertyMock.side_effect = [False, True]`),
  (2) timeout when reconnect never completes, and
  (3) the public `export_storage_state` waiting for reconnect
      end-to-end.
  The `RECONNECT_WAIT_TIMEOUT = 0.2` override keeps the test fast.
  Good shape.

- **Potential follow-up** (not blocking): `cdp_client` calls outside
  these four helpers (e.g. `Network.clearBrowserCookies` on line
  1355) bypass the guard. In particular, `clear_cookies()` on line
  1354 calls `self.cdp_client.send.Network.clearBrowserCookies()`
  directly — that's the **public** API and *does* need the guard,
  but the diff only patches `_cdp_clear_cookies` (the underscore-
  prefixed internal). Either wire the guard into `clear_cookies()`
  too, or delete the public method in favor of routing all callers
  through the underscore variant.

## Rationale

This is a real fix for a real race: storage helpers were happy to
fire `Storage.getCookies` against a CDP transport mid-reconnect and
get back a hung `wait_for` or a stale session. The guard is small,
the error semantics are right, and the tests cover the three relevant
branches. The one nit worth catching before merge is that the public
`clear_cookies()` (line 1354) bypasses the guard while the private
`_cdp_clear_cookies` (line 3370) is protected — that's an
inconsistency the next user will report.

## What I learned

When a session-style class has both a "public API method" and an
"internal `_cdp_xxx` helper" doing the same thing, retrofitting a
guard onto only the internal helpers is a classic source of
follow-up bugs. The right move is either to consolidate (all callers
go through the helper) or to apply the guard at both layers. This PR
is one re-routing away from a complete fix.
