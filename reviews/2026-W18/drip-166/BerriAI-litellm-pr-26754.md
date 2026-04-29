# BerriAI/litellm #26754 — [Fix] /v1/messages — plumb client-supplied timeout to httpx call

- **PR:** https://github.com/BerriAI/litellm/pull/26754
- **Head SHA:** `e6b7bc0041ebe20d8733f2a0de7269287aaa1b1c`
- **Size:** ~+240 / −0 (≈40 lines of production plumbing across 2 files, ~200 of focused tests)

## Summary
The `/v1/messages` (Anthropic experimental pass-through) path was silently dropping per-request `timeout` / `stream_timeout` — the router populated `litellm_params.timeout`, but the wrapper at `litellm/llms/anthropic/experimental_pass_through/messages/handler.py` never read it, so `httpx` always fell back to its global default (~5 minutes). Fix reads `litellm_params.stream_timeout if stream else litellm_params.timeout`, coerces `str → float`, and threads it through three layers: `anthropic_messages_handler` → `async_anthropic_messages_handler` → `_async_post_anthropic_messages_with_http_error_retry` → `async_httpx_client.post(timeout=...)`. Mirrors the existing `Router._get_timeout` pattern so direct (non-router) callers get the same stream/non-stream timeout selection.

## Specific observations

1. **The load-bearing read at `litellm/llms/anthropic/experimental_pass_through/messages/handler.py:454-462`:**
   ```python
   _resolved_timeout = (
       litellm_params.stream_timeout
       if stream and litellm_params.stream_timeout is not None
       else litellm_params.timeout
   )
   if isinstance(_resolved_timeout, str):
       _resolved_timeout = float(_resolved_timeout)
   ```
   Three correct decisions in seven lines: (a) stream-vs-non-stream selection mirrors `Router._get_timeout`; (b) the `is not None` check on `stream_timeout` correctly falls back to `timeout` when only the latter was set (without it, a `stream=True` call with only `timeout=` set would get `None` instead of the configured timeout); (c) `str → float` coercion is needed because `GenericLiteLLMParams.timeout` is typed `Union[float, str]` (some YAML configs land here as strings). The `httpx`-level call expects `float | httpx.Timeout`, so the coercion has to happen before the boundary, not at it.

2. **Three test cases at `tests/test_litellm/llms/anthropic/.../test_anthropic_experimental_pass_through_messages_handler.py:499-583`** cover exactly the three semantic axes:
   - `test_..._passes_litellm_params_timeout_to_base_handler` — the basic forwarding (asserts `mock_base.call_args.kwargs["timeout"] == 0.5`)
   - `test_..._coerces_string_timeout_to_float` — the type-coercion case (asserts both value and `isinstance(..., float)`)
   - `test_..._prefers_stream_timeout_for_streaming_calls` — the stream selection case (asserts `stream_timeout=0.5` wins over `timeout=10.0` when `stream=True`)

   Plus four integration-shaped tests in `tests/test_litellm/llms/custom_httpx/test_llm_http_handler.py:323-460` that walk the full chain (`async_anthropic_messages_handler` → retry helper → `httpx_client.post`) parametrized on `stream` ∈ {False, True}. The `defaults_timeout_to_none_when_unset` test at `:421-460` correctly pins backward-compat: callers that don't supply a timeout get `kwargs["timeout"] is None`, preserving `AsyncHTTPHandler`'s own default fallback. This is the right shape.

3. **Three plumbing additions in `litellm/llms/custom_httpx/llm_http_handler.py`:**
   - `_async_post_anthropic_messages_with_http_error_retry` signature gains `timeout: Optional[Union[float, httpx.Timeout]] = None` (`:1836`)
   - `async_anthropic_messages_handler` signature gains the same parameter at `:1906` and forwards at `:2044`
   - sync `anthropic_messages_handler` gains and forwards at `:2111` and `:2134`

   All three are `Optional[...] = None` defaulted, so this is API-additive, not breaking. Existing callers continue to work.

4. **`stream` passed alongside `timeout` to `httpx_client.post`** in the retry helper at `:1851` — both are forwarded together via `kwargs`. The `stream=stream is bool` distinction in `httpx` versions matters because pre-0.27 `httpx` interpreted `stream=True` differently from later versions. Worth verifying the `aiohttp_transport` / `httpx_transport` selection is honored — but this PR doesn't touch transport selection so any pre-existing transport bug is unaffected.

5. **Sync vs async asymmetry:** the sync `get()` adapter (mentioned in PR title context) coerces `httpx.TimeoutException → litellm.Timeout`, but the async one does not. This PR doesn't fix or change that — but since the new `timeout=...` parameter makes the timeout boundary observable, an unhandled `httpx.TimeoutException` on the async path will now actually fire (instead of being lost in the 5-minute default). Worth a follow-up to add the same `except httpx.TimeoutException: raise litellm.Timeout(...)` shim on the async path.

## Risks

- **Behavioral change for any caller that was relying on the 5-minute default** by setting a low `timeout` via `litellm_params` and observing it being silently ignored. Their requests will now time out at the configured value. This is the *correct* behavior, but it's a behavioral change that should be release-noted.
- **String-form timeout from YAML** (e.g. `timeout: "30"` in `config.yaml`) would, prior to this PR, have produced a `TypeError` at the `httpx`-level cast — silently caught by the `except Exception` somewhere upstream. Post-PR, `float("30") == 30.0` works correctly. But `timeout: "30s"` (with units) would now raise `ValueError: could not convert string to float: '30s'` at the wrapper, not at httpx. Bigger surface for malformed-config errors. Worth a `try/except ValueError: log + fall through to None` if YAML-with-units is a real configuration users do.
- **Async `httpx.TimeoutException → litellm.Timeout` not adapted** — the new "timeout actually fires" reality on the async path means callers will see the raw `httpx.TimeoutException` instead of `litellm.Timeout`. Code paths that catch `litellm.Timeout` for retry/fallback will be silently bypassed.
- **`_resolved_timeout` is unconditional even when `litellm_params.timeout is None`** — in that case the helper coerces `None` (which is not `isinstance(_, str)`, so the coerce branch is skipped, and `None` propagates). Correct behavior, but reading the code top-down it's easy to miss that `None` flows through fine.

## Suggestions

- **(Recommended)** Adapt `httpx.TimeoutException → litellm.Timeout` on the async path inside `_async_post_anthropic_messages_with_http_error_retry`, mirroring the sync `get()` shim. Otherwise post-merge users will see a new exception class they don't catch.
- **(Recommended)** Test a malformed string timeout (`"30s"`) and decide on the policy — either fail fast with a clear error message naming the config key, or `try/except ValueError` and fall back with a warning.
- **(Recommended)** Release-note the behavior change: "/v1/messages now honors per-request `timeout` / `stream_timeout` from `litellm_params`. Previously these were silently ignored and requests defaulted to httpx's ~5-minute timeout."
- **(Optional)** A grep for other `experimental_pass_through` paths (e.g. `/v1/embeddings` pass-through if it exists) — same bug class likely lurks anywhere a wrapper hands off to `BaseLLMHTTPHandler` without forwarding `timeout`.

## Verdict: `merge-after-nits`

Correct silent-failure fix at the right layer (read at boundary, coerce before httpx call), with strong test coverage on all three semantic axes (forwarding, coercion, stream selection) plus integration tests that pin the full plumbing chain. The nits are async-side `TimeoutException` adaptation and one malformed-string-input test.

## What I learned

The bug class is "the field exists in the dataclass but no consumer reads it." This is invisible to type checkers (the field is correctly typed) and invisible to integration tests that use the default timeout (the difference is only observable at long-tail latencies). The right detection is contract testing — assert that *every* consumer of `litellm_params` reads each documented field that's relevant to its call shape. This PR's three-axis test coverage (forwarding + coercion + selection) is the canonical shape for plumbing fixes.
