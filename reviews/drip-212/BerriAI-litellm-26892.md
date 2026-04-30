# BerriAI/litellm PR #26892 — fix: forward /v1/messages request timeout

- Repo: `BerriAI/litellm`
- PR: https://github.com/BerriAI/litellm/pull/26892
- Head SHA: `2fd663ad0a732b72b8755b87615a195cc57dd988`
- State: OPEN
- Files: 2 (`litellm/llms/custom_httpx/llm_http_handler.py`, `tests/test_litellm/llms/custom_httpx/test_llm_http_handler.py`), +101/−1
- Verdict: **merge-after-nits**

## What it does

Threads per-request timeout settings through the Anthropic-compatible
`/v1/messages` POST path that previously ignored them. Before, in
`BaseLLMHTTPHandler._async_post_anthropic_messages_with_http_error_retry`
at `litellm/llms/custom_httpx/llm_http_handler.py:1903-1915`, the call
to `async_httpx_client.post(...)` did not pass a `timeout=` kwarg —
meaning the underlying `AsyncHTTPHandler` fell back to the global
client default, regardless of `litellm_params.timeout`,
`litellm_params.stream_timeout`, or
`litellm_params["request_timeout"]`. This silently broke per-route
timeout overrides set on Anthropic `/v1/messages` proxy routes.

The fix introduces two new statics on `BaseLLMHTTPHandler`:

1. `_coerce_http_timeout(timeout)` (line 184-204) — accepts
   `None | float | int | str | httpx.Timeout`, returns
   `None | float | httpx.Timeout`. Recognizes `"os.environ/..."` strings
   and resolves them through `litellm.get_secret`. Logs a warning and
   returns `None` on `TypeError`/`ValueError` rather than throwing,
   which is the right shape for "timeout is best-effort, never break
   the request just because the env var was malformed."
2. `_get_anthropic_messages_timeout(*, litellm_params, stream)` (line
   206-217) — precedence:
   - if `stream and litellm_params.stream_timeout is not None` →
     `stream_timeout`
   - elif `litellm_params.timeout is not None` → `timeout`
   - else → `dict(litellm_params).get("request_timeout")`

Then computes the timeout once outside the retry loop and passes it
on every attempt:

```python
timeout = self._get_anthropic_messages_timeout(
    litellm_params=litellm_params,
    stream=stream,
)
for attempt_idx in range(max_attempts):
    response = await async_httpx_client.post(
        ...,
        timeout=timeout,
    )
```

Tests are 4 parametrized cases pinning each precedence arm + 2
explicit cases for env-var resolution (valid + invalid).

## What's right

- The precedence order — `stream_timeout` (only when streaming),
  then `timeout`, then `request_timeout` — matches what the rest of
  the codebase does for streaming endpoints, so there's no new
  semantic to learn here.
- Coercion is a separate static, not inlined into the precedence
  function. That's the right shape because the env-var resolution and
  the float-coerce-with-graceful-fallback logic is reusable across the
  3 input fields and is unit-testable in isolation (the two
  `test_coerce_http_timeout_*` cases hit it directly without going
  through the retry helper).
- Computed *outside* the retry loop. If a request retries 3× due to
  HTTP errors, the timeout is computed once. Without that, a malformed
  env-var would log a warning per attempt.
- The "warning + return `None`" branch on coerce failure is correct:
  letting `httpx` use its built-in default is strictly safer than
  raising, since the user's intent (override the default) was
  malformed but their underlying intent (make the request) is fine.
- 4 parametrized tests cover all 3 precedence arms (timeout w/o
  stream, timeout w/ stream, stream_timeout w/ stream wins,
  request_timeout fallback). Plus 2 env-var coerce tests. Solid
  coverage.

## Nits (request before merge)

1. **`stream and litellm_params.stream_timeout is not None`** — note
   the `stream` is the *boolean* arg passed in, not derived from the
   request body. Worth a comment that `stream_timeout` is only
   honored when the *caller* declared streaming, not when the request
   body sets `"stream": true`. The two can disagree in practice.
2. **Missing test: `stream=True, stream_timeout=None, timeout=X`.**
   The 4 parametrized cases don't cover the case where streaming is
   on but `stream_timeout` is unset — the precedence falls through to
   `timeout`, which is correct, but the test matrix doesn't pin it.
   One more parametrized row would close the gap.
3. **Type annotation drift.** The new
   `_coerce_http_timeout(timeout: Optional[Union[float, int, str,
   httpx.Timeout]])` returns `Optional[Union[float, httpx.Timeout]]`
   — but `int` inputs go through `float(cast(Union[float, int, str],
   timeout_value))` and emerge as `float`. Either include `int` in
   the return union or document that ints are normalized to float
   (subtle but matters for callers that compare with `is`/`==` on a
   typed config).
4. **`os.environ/` vs `OS.ENVIRON/` etc.** The check is
   `timeout.startswith("os.environ/")` — case-sensitive. If the rest
   of the codebase has a centralized "is this an env-var ref?" helper
   (the typical `is_secret_str(...)` shape), use it for consistency
   instead of inlining the prefix check. Otherwise, deserves a
   one-line comment naming the prefix convention.
5. **Test bleed.** `test_coerce_http_timeout_reads_env_secret` and
   `test_coerce_http_timeout_returns_none_for_invalid_env_secret` both
   use `with patch("litellm.get_secret", ...)` rather than a fixture
   — fine for two tests, but if more cases get added (e.g. `None`
   from `get_secret`), a fixture would be cleaner.
6. **No assertion that `timeout` is not re-coerced per attempt.**
   The "computed once outside the loop" property is the whole point of
   placing the assignment at line 1906; a quick test that mocks
   `_get_anthropic_messages_timeout` and asserts `call_count == 1`
   even with `max_retry == 3` would lock the perf invariant.

## Why merge-after-nits

This fixes a real per-request-config-ignored bug with the right
two-level decomposition (coerce / select), the test matrix covers
the precedence cleanly, and the fail-soft-on-coerce-error policy
matches expectations for a best-effort override. Nits are
documentation, type-annotation tightening, and one missing matrix
cell — nothing blocks correctness, but landing them keeps the
contract auditable.
