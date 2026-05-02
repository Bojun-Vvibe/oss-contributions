# Review — BerriAI/litellm#27045

- PR: https://github.com/BerriAI/litellm/pull/27045
- Title: fix(openai): recover from SSE response when stream=false (#25766)
- Head SHA: `ab1d2d0b337827ed2f611dde055ad3134b5fcbdf`
- Size: +211 / −2 across 3 files
- Verdict: **needs-discussion**

## Summary

Some OpenAI-compatible upstreams (issue #25766 cites Qwen-flavoured
endpoints) ignore `stream=false` and always reply with an SSE body.
The current openai client path detects this at
`make_openai_chat_completion_request` (the parsed response lacks
`model_dump`) and raises `OpenAIError(500, "Empty or invalid
response from LLM endpoint…")`. This PR adds a recovery path: a new
`try_parse_sse_response_body` helper in
`litellm/llms/openai/common_utils.py` parses the raw text body,
strips `data:` prefixes via
`CustomStreamWrapper._strip_sse_data_from_chunk`, drops `[DONE]`,
JSON-decodes each chunk, and feeds them into
`litellm.stream_chunk_builder` to assemble a single
`ModelResponse`. If recovery succeeds, the call returns a normal
synchronous response; if it fails (no SSE lines, builder raises,
result isn't a `ModelResponse`), the original 500 still fires.

## Evidence

- `litellm/llms/openai/common_utils.py:294-342` — new helper. Notable
  details: it imports `CustomStreamWrapper` lazily to avoid a
  circular import; it returns `None` on every failure mode and
  swallows JSONDecodeError per line; final type-guard ensures the
  caller always gets a `ModelResponse` or `None`.
- `litellm/llms/openai/openai.py:447-455` (async path) and
  `litellm/llms/openai/openai.py:485-495` (sync path) — both wrap
  the existing "no model_dump" branch with a `try_parse_sse_response_body(
  getattr(raw_response, "text", None))` attempt before raising the
  500.
- Tests in `tests/test_litellm/llms/openai/test_openai_empty_response.py`
  cover the sync path (`test_sync_sse_response_recovers`, asserts
  `"Hello world"` reassembled across two chunks) and the async path
  (`test_async_sse_response_recovers`). Existing
  `test_*_streaming_response_passes_through_without_model_dump`
  tests still pass because the recovery only fires when `data.get(
  "stream")` is falsy.

## Concerns / why "needs-discussion"

- **Silent semantic change.** Today, a non-streaming caller that
  receives an SSE body gets an explicit 500 they can route through
  fallbacks. After this lands, the same upstream returns a 200 with
  a synthesised response. That changes the error budget for
  affected models: callers who relied on the 500 to drive failover
  to a healthier deployment will silently keep talking to a broken
  upstream forever. This is a feature for some users and a
  regression for others; consider gating behind
  `litellm.recover_sse_on_non_stream` or a per-model flag.
- **Tool / structured-output handling.** `stream_chunk_builder`
  reassembles tool_calls and content, but if the SSE chunks include
  `function_call` deltas the recovery path is not exercised by the
  current tests. An upstream that mis-streams a tool-using
  completion could land malformed `tool_calls` in the synthesised
  response. Adding a tool-call test case before merging would close
  this.
- **Unbounded body parsing.** `body.splitlines()` on a multi-MB
  buffered SSE response will allocate the whole list eagerly. Not
  catastrophic — it's bounded by the upstream's response size — but
  worth noting if an upstream streams tens of thousands of tokens.
- **Error attribution.** When `stream_chunk_builder` raises, the
  helper logs at `verbose_logger.debug` and falls through to the
  original 500. The user-facing 500 will say "Empty or invalid
  response" even though the *real* failure was builder-side. A
  dedicated error message ("recovered SSE body but
  stream_chunk_builder failed: …") would be much more debuggable.

Worth a maintainer discussion on the gating question before merging.
The recovery itself is well-implemented; the policy question is
whether silent recovery is the right default.
