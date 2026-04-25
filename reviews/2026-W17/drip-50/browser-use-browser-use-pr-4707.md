---
pr: 4707
repo: browser-use/browser-use
sha: def77cfd7527b4e2961cfb7f7aaf9b596c90ad1c
verdict: merge-as-is
date: 2026-04-25
---

# browser-use/browser-use#4707 — test: cover BrowserSession.close before start

- **URL**: https://github.com/browser-use/browser-use/pull/4707
- **Author**: cuyua9

## Summary

A test-only PR adding one regression test
(`test_close_before_start_is_safe`) to
`tests/ci/browser/test_session_start.py`, asserting that calling
`BrowserSession.close()` on a session that has never been started
is safe and has three observable post-conditions: (a)
`_cdp_client_root is None`, (b) `event_bus is not None`, (c) the
`event_bus` was *replaced* (rebuilt) rather than left as the
original instance.

## Reviewable points

- `tests/ci/browser/test_session_start.py:61` — the new test:

  ```py
  async def test_close_before_start_is_safe(self, browser_session):
      original_event_bus = browser_session.event_bus
      await browser_session.close()
      assert browser_session._cdp_client_root is None
      assert browser_session.event_bus is not None
      assert browser_session.event_bus is not original_event_bus
  ```

  The third assertion (`is not original_event_bus`) is the
  interesting one — it's pinning a non-obvious side-effect of
  `close()`: that even when there was nothing to close, calling
  `close()` rotates the event bus to a fresh instance. That's
  the contract that lets the same `BrowserSession` object be
  reused (`close()` then `start()`) without leaking subscribers
  from before. Without this assertion you can imagine a future
  optimization that says "if `_cdp_client_root is None`, return
  early from close()" and silently breaks reuse.

- The test is placed inside the existing
  `TestBrowserSessionStart` class right after
  `test_start_already_started_session` (line 55), which is the
  natural neighbor for "what does the lifecycle look like in
  unusual orderings" coverage. Good location.

- No production code changed, no new imports needed beyond the
  existing fixture `browser_session`. The PR description notes
  the test file's header already listed this path as expected
  coverage — i.e. a documented gap is being closed, not new
  surface invented.

- Minor: `original_event_bus` is captured but the test doesn't
  assert anything about its post-close state (e.g. whether its
  subscribers were drained, whether `original_event_bus` itself
  is in a "closed" state). Probably out of scope — the contract
  being locked here is "the *session's* event_bus is rotated",
  not "the old bus is shut down". If a future test wants to lock
  the latter, that's a separate test.

- The auto-generated cubic summary in the PR body claims a
  pr-value of 89/100. That's marketing noise; the real value is
  that this is a 10-line atomic test that locks one specific
  contract (event-bus rotation on no-op close) which is the kind
  of property that's *very* easy to accidentally regress in a
  future "close() should be cheap when there's nothing to close"
  optimization.

## Rationale

Test-only, atomic, locks a concrete and non-obvious contract
(event-bus rotation), uses the existing fixture, lives in the
right test class. Nothing to push back on. The third
assertion in particular is the one I'd want present before any
future maintainer "optimizes" `close()` to short-circuit on an
unstarted session.

## What I learned

The most useful regression tests are usually the ones that lock
*non-obvious* side-effects of supposedly-no-op operations —
"close() on an unstarted session does nothing" sounds like a
trivial property, but the *event-bus-is-rotated* part is the
non-obvious bit that would silently break session-reuse if a
future cleanup decided that "nothing to do" means "return early".
For any object with lifecycle methods (`start`/`stop`,
`open`/`close`, `connect`/`disconnect`), the high-value tests are:
(a) lifecycle-ordering edges (close-before-start,
double-start, double-close), (b) post-condition observability
(what does the object look like after each transition, even the
no-op ones), and (c) reuse safety (can the same object be
re-driven through the lifecycle?). This PR lands (a) and (b) for
one specific case and implicitly protects (c).
