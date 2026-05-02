# BerriAI/litellm PR #27045 — fix(openai): recover from SSE response when stream=false

- URL: https://github.com/BerriAI/litellm/pull/27045
- Head SHA: `423bf2d2c962217ae8016bcf39b2319321c9fb31`
- Fixes: #25766
- Verdict: **merge-after-nits**

## What it does

Some OpenAI-compatible reverse proxies (the bug report cites `dialagram.me/router`)
ignore `stream=false` and always reply with an SSE body. Previously
`make_(sync|async)_openai_chat_completion_request` would raise
`OpenAIError(500, "Empty or invalid response from LLM endpoint...")`. This
PR adds a `try_parse_sse_response_body` helper that, on a parse miss,
attempts to (a) split the raw text body by `\n`, (b) strip `data: ` prefixes
via `CustomStreamWrapper._strip_sse_data_from_chunk`, (c) `json.loads` each
chunk, (d) aggregate via `litellm.stream_chunk_builder`, and (e) return a
proper `ModelResponse`. Falls back to the original error on any failure.

## Specifics I checked

- `litellm/llms/openai/common_utils.py:288-335` — new helper. The early
  returns on empty body / non-string body / non-JSON line / no chunks are
  defensive in the right places. Returns `None` when nothing could be
  recovered, which the caller treats as "fall through to the existing
  error".
- `litellm/llms/openai/openai.py:447-456` and `:485-498` — the recovery
  hook is added in both async and sync paths, gated by `if not data.get("stream")
  and not hasattr(response, "model_dump")`. Same guard as the existing error,
  so behavior only changes inside that error window. Good scoping.
- Tests in `tests/test_litellm/llms/openai/test_openai_empty_response.py`
  cover (1) sync SSE recovery, (2) async SSE recovery, (3) garbage non-SSE
  text still raises. The garbage-text test is the most important guard —
  prevents the helper from masking real upstream failures.

## What I like

- Defensive: SSE parse failure preserves the *exact* legacy error message,
  so users currently working around this bug don't see new failure modes.
- Reuses `CustomStreamWrapper._strip_sse_data_from_chunk` and
  `litellm.stream_chunk_builder` rather than writing a parallel SSE parser.
  Consistency with the streaming path is a good property.

## Nits

1. **`raw_response.text`.** The recovery reads `getattr(raw_response, "text", None)`.
   For the `httpx.Response` wrapped by openai-python's raw-response
   helpers, `.text` is a property that triggers a body read. If the
   response stream has already been consumed by `parse()` returning a
   non-`ModelDump`-able object, `.text` may be empty or raise. A brief
   comment explaining this is safe (or a `try/except` around the
   `getattr`) would help future maintainers.
2. **`_strip_sse_data_from_chunk` is a private method.** Calling
   `CustomStreamWrapper._strip_sse_data_from_chunk` from outside the
   class makes that method a de-facto public API. Either expose it
   formally or inline the trivial `line.removeprefix("data: ")` logic.
3. **Test data uses `qwen-3.6-plus`** as the model name. That date
   (2026-05) seems forward-looking; it works, but if you want the test
   to age well, use a stable name like `gpt-3.5-turbo`.

## Risk

Low. The only path that changes behavior is the previously-erroring
"non-streaming response with no `model_dump`" branch, and unrecoverable
inputs still raise the same exception. Verdict `merge-after-nits` for the
private-method call and the `text` property safety note.
