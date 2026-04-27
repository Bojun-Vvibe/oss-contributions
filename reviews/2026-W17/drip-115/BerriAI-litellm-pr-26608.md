# BerriAI/litellm#26608 — respect timeout config in Anthropic adapter

- PR: https://github.com/BerriAI/litellm/pull/26608
- Head SHA: `ec91aa7c`
- Diff: +278 / -0 across `litellm/llms/custom_httpx/llm_http_handler.py` (+4) and a new `tests/test_litellm/proxy/pass_through_endpoints/test_anthropic_timeout.py` (+274)

## What it does
Threads `timeout` from `kwargs` into the Anthropic `/v1/messages` adapter path:
- `llm_http_handler.py:1836` — adds `timeout: Optional[Union[float, httpx.Timeout]] = None` parameter to `_async_post_anthropic_messages_with_http_error_retry`.
- `:1851` — forwards it into `async_httpx_client.post(...)`.
- `:1921` — `async_anthropic_messages_handler` extracts `timeout = kwargs.get("timeout", None)` and passes it down at `:2044`.

This matches the established pattern: Router places the timeout into `kwargs` via `_update_kwargs_with_deployment` (existing infra), and other handlers (responses, completion, embedding) already pluck it from there. The Anthropic adapter was the odd one out, silently inheriting httpx's 600s default no matter what `--request_timeout` / `litellm_settings.request_timeout` / `router_settings.timeout` said.

## Strengths
- Tight, surgical fix to a real silent-default footgun. The 600s default is exactly the kind of thing that shows up only when a model hangs in production.
- Five new tests exercise the right matrix:
  - non-streaming short timeout → expects timeout error
  - non-streaming sufficient timeout → expects success
  - streaming short timeout → expects timeout error
  - streaming sufficient timeout → expects SSE chunks
  - streaming with `stream_timeout=1` and `timeout=30` → asserts `stream_timeout` precedence
- Tests use `unittest.mock.patch` on `AsyncHTTPHandler.post` (`MOCK_TARGET` constant) — fast, no actual network, deterministic.

## Concerns
- The tests file is +274 lines for a +4-line behavior change — that's a healthy ratio, not a complaint, but skim the test for setup duplication. If `_mock_anthropic_response()` and `_mock_streaming_response()` already exist elsewhere in `tests/test_litellm/proxy/pass_through_endpoints/`, sharing them via a helper module avoids drift.
- `timeout = kwargs.get("timeout", None)` at `:1921` doesn't validate the type. If a user passes `timeout="10"` (string) via config, httpx will raise. Other handlers in the file may already coerce; worth verifying consistency. Either way, a one-line `if isinstance(timeout, str): timeout = float(timeout)` would harden this.
- The `stream_timeout` precedence test is good, but the actual code change in this PR doesn't show stream_timeout handling — that must already exist in `async_httpx_client.post` or in the streaming path. PR description mentions "streaming requests honour the same timeout for the initial connection" but the diff doesn't visibly route `stream_timeout`. Confirm the test isn't asserting behavior of unrelated code.
- One Pre-Submission checkbox unchecked: "My PR passes all unit tests on `make test-unit`". Worth confirming before merge.

## Verdict
**merge-after-nits** — fix is right, tests are real. Clarify the stream_timeout codepath in the PR description or diff, and run `make test-unit`, then ship.
