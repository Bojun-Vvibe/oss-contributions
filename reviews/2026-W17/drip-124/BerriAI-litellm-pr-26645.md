# BerriAI/litellm #26645 — feat(logging): add retry settings for generic API logger

- **PR**: [BerriAI/litellm#26645](https://github.com/BerriAI/litellm/pull/26645)
- **Head SHA**: `6d6be44b`
- **Stats**: +205/-11 across 4 files

## Summary

Adds three new opt-in fields to `GenericAPILogger.__init__` —
`max_retries: int = 0`, `retry_delay: float = 1.0`,
`timeout: Optional[Union[float, httpx.Timeout]] = None` — and threads
them through `LoggingCallbackManager._add_custom_callback_generic_api_str`
so they're configurable via `litellm.callback_settings[name]`. New
`_post_with_retries` helper at
`generic_api_callback.py:262-285` wraps the previous bare
`async_httpx_client.post(...)` call with an attempt loop that retries
transient errors (litellm timeouts, httpx transport errors, HTTP 5xx)
with exponential backoff and re-raises non-retriable errors (4xx, etc.)
immediately. Replaces both call sites in `async_send_batch` (parallel
per-entry mode at `:386` and batched-payload mode at `:413`) so both
log-format paths get the same retry semantics.

## Specific findings

- `generic_api_callback.py:240-247` — `_should_retry_exception` is the
  load-bearing classifier: `isinstance(exception, (litellm.Timeout,
  httpx.TransportError))` returns true (`TransportError` is the
  parent of `ConnectTimeout`/`ReadTimeout`/`ConnectError`/`PoolTimeout`
  so the single isinstance covers the full transient-network set), and
  `httpx.HTTPStatusError` only retries when
  `response.status_code >= 500` so 4xx auth/permission/payload errors
  short-circuit immediately. Correct policy — retrying a 401 on a
  callback endpoint will never recover and just delays the (correct)
  visible failure.
- `:249-254` — `_sleep_before_retry` early-returns when
  `retry_delay <= 0` (guard for the `retry_delay=0` test config that
  would otherwise spin a no-op `asyncio.sleep(0)` per attempt) and
  computes `delay = retry_delay * (2**attempt)` for true exponential
  backoff. Note that `attempt` starts at 0 → first retry waits exactly
  `retry_delay` seconds, second waits `2*retry_delay`, etc. Matches
  conventional exponential-backoff semantics; no jitter, which for a
  single-instance batch logger is acceptable but worth flagging
  if multiple replicas share the endpoint (thundering herd risk).
- `:256-285` — `_post_with_retries` builds `post_kwargs` once outside
  the loop including `url`/`headers`/`data` (and `timeout` only if
  configured, so `None` doesn't accidentally override the httpx
  client's default), then loops `total_attempts = self.max_retries + 1`
  times. Critical contract: `is_last_attempt = attempt == self.max_retries`
  with `if is_last_attempt or not should_retry: raise` re-raises the
  original exception unchanged on the last attempt or any non-retriable
  error — preserves stack traces and downstream error-classifier
  behavior. The fall-through `raise RuntimeError("Generic API Logger
  retry loop exited unexpectedly")` at `:285` is unreachable defensive
  code (every iteration either returns or re-raises) but correct
  belt-and-suspenders for type checkers expecting the function to
  return.
- `logging_callback_manager.py:224-243` — settings read via
  `callback_config.get("max_retries", 0) or 0` then `max(0, int(...))`
  clamps negative or non-numeric inputs to 0, and `retry_delay` is
  similarly clamped to `>= 0.0` via the
  `max(0.0, float(0.0 if value is None else value))` pattern. The
  cache key now includes `max_retries`/`retry_delay`/`timeout` at
  `:246-248` so a config change forces logger re-instantiation rather
  than silently reusing the prior settings.
- `:222-228` — defaults preserve backward compatibility: `max_retries=0`
  means *exactly one attempt with no retries*, matching pre-PR
  behavior. Existing deployments that don't set `max_retries` are
  unaffected.
- Test coverage at `test_generic_api_callback.py:474-566` is thorough:
  three tests for the three branches of the classifier
  (`_retries_timeout_then_succeeds` for `litellm.Timeout`,
  `_retries_5xx_then_succeeds` for `HTTPStatusError(503)`,
  `_does_not_retry_4xx` for `HTTPStatusError(401)` asserting
  `mock_post.assert_called_once()`). The 4xx test is the most
  important one — pins that the policy correctly *doesn't* retry
  auth errors. Plus integration test in
  `test_logging_callback_manager.py:369-405` that pins the
  `callback_settings` → `GenericAPILogger` field plumbing.

## Nits / what's missing

- No jitter in the backoff. For a single-pod proxy this is fine, but
  for multi-replica deployments a thundering-herd scenario after a
  brief callback-endpoint outage could amplify load. Worth a follow-up
  to add optional `retry_jitter: float = 0.0`.
- The unreachable `raise RuntimeError(...)` fall-through at `:285`
  defends against type checkers but reads as a "this can happen"
  signal to readers. Adding a `# unreachable: every iteration returns
  or raises` comment would prevent confusion.
- `_post_with_retries` operates on a single post; in the parallel-mode
  call site at `:386` retries are per-entry so a slow endpoint can
  cumulatively block the batch send for `(max_retries + 1) * timeout *
  num_entries` worst-case. The previous behavior was strict bounded
  parallelism with `gather` — keeping that property under retries
  would need a bounded semaphore. Not blocking but worth a comment in
  the docstring on the parallel-mode interaction.
- No retry metric / counter exposed. A `retry_count` field on the
  logger that operators can scrape would help diagnose
  endpoint-flakiness vs config-bug situations.

## Verdict

**merge-as-is** — feature is opt-in (`max_retries=0` default
preserves prior behavior), classifier policy is conservative and
correct (transient errors only), test coverage exercises all three
classifier branches plus the config-plumbing path, and the cache-key
update prevents stale-logger reuse across config edits. Nits are
follow-up improvements rather than merge blockers.
