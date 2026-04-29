# BerriAI/litellm #26750 — Cursor-based stream resume for `GET /v1/responses/{id}`

- **PR:** https://github.com/BerriAI/litellm/pull/26750
- **Size:** +619 lines net (production + tests)

## Summary
Adds support for the OpenAI Responses API's cursor-based stream resume — `client.responses.retrieve(id, stream=True, starting_after=N)` — which the OpenAI Python SDK sends as `GET /v1/responses/{id}?stream=true&starting_after=N`. Previously the proxy silently dropped the query-string params and returned the cached non-streamed response, breaking SDK-driven resume.

## Specific observations

1. **Root cause is at the proxy ingress** in `litellm/proxy/proxy_server.py`, the `get_response` handler around lines 400-422. The pre-fix code did `data = await _read_request_body(request=request); data["response_id"] = response_id` and called the processor — but `_read_request_body` is a no-op for GET (no body to read), so `stream` and `starting_after` from the query string never made it into `data`. The fix reads them explicitly off `request.query_params` and merges them into the dict, with type coercion (`stream` → bool via lower-case `("true", "1")` check, `starting_after` → `int(...)`).

2. **400 on malformed `starting_after`** at line 416-422 is the right call: catching `ValueError` from `int(query_starting_after)` and raising `HTTPException(status_code=400, detail="Query parameter 'starting_after' must be an integer sequence number, got {query_starting_after!r}.")`. Without this, the bad value would propagate downstream and surface as either a 500 or a confusing provider error. The error message includes the bad value via `!r` which is friendly for SDK debugging.

3. **Provider transform layer:** `litellm/llms/azure/responses/transformation.py:249-275` adds `stream: bool = False, starting_after: Optional[int] = None` to `transform_get_response_api_request`, and conditionally puts them into the returned `data` dict (`data["stream"] = "true"` as a string, `data["starting_after"] = starting_after` as int). The string-"true" choice matches what the OpenAI/Azure endpoints expect on the wire — they parse query params as strings, and `True`-the-Python-bool would serialize to `"True"` (capital T) which Azure rejects. Good attention to that detail.

4. **Base class signature update** at `litellm/llms/base_llm/responses/transformation.py:165-179` adds the same two kwargs with sensible defaults. The docstring explicitly tells subclasses: "Implementations that do not support resume should ignore these arguments. Implementations that do support resume should put `stream` and `starting_after` into the returned `data` dict so the underlying HTTP layer attaches them as query parameters." This is the right contract — opt-in by default, no-op if a provider can't resume.

5. **HTTP handler streaming path** at `litellm/llms/custom_httpx/http_handler.py:486-502` and `1027-1080` adds a `stream: bool` parameter to both async and sync `get()`. The sync path is more careful — it wraps the `client.send(req, stream=stream, ...)` call in a try/except handling `httpx.TimeoutException → litellm.Timeout` and `httpx.HTTPStatusError → _raise_masked_sync_error(e, stream)`. The async path does not wrap with the same exception adapter — that's an asymmetry worth flagging. Streaming gets typically have longer-tail timeouts than non-stream gets, so a stream-aware timeout adapter on the async side would be useful.

6. **Iterator-vs-response return type** at `llm_http_handler.py:2624-2660`: `get_responses` now returns `Union[ResponsesAPIResponse, BaseResponsesAPIStreamingIterator, Coroutine[..., Union[ResponsesAPIResponse, BaseResponsesAPIStreamingIterator]]]`. The triple-arm union is hard to read but accurate. The `aget_responses` / `get_responses` wrappers in `litellm/responses/main.py:1365-1547` propagate `stream` and `starting_after` cleanly.

7. **Volcengine and Manus `transformation.py` updated to accept the new kwargs but ignore them** (test at `test_get_responses_stream_resume.py` lines 733-749 pins this contract). This is the right shape — the base-class signature change forces subclasses to explicitly opt out by accepting-and-ignoring rather than silently failing on `TypeError: unexpected keyword argument`.

8. **Endpoint test coverage is good:**
   - `test_get_response_forwards_stream_resume_query_params` (line 574) — happy path, asserts `captured["data"] == {"response_id": "resp_test123", "stream": True, "starting_after": 6}`.
   - `test_get_response_rejects_non_integer_starting_after` (line 614) — error path, asserts 400 + exact error message.

   Missing: a test for `stream=false` or `stream=0` to confirm the negative case stays out of `data`. The current implementation only checks `("true", "1")` so `"false"`/`"0"`/`""` all correctly drop the key, but a test would lock that in.

## Risks

- **Sync vs async exception-handling asymmetry** in `http_handler.py` — sync path adapts `httpx.TimeoutException` to `litellm.Timeout`, async path doesn't. Could leak raw httpx exceptions to async callers.
- **String vs int wire format for `stream`** — Azure expects `"true"`, but if a future provider expects a JSON-bool `true` (rare), the per-provider transform handles it correctly via the `data` dict. OK.
- **Caller resource-management contract** for the streaming get is documented in the docstring at `http_handler.py:494-499` ("Caller is responsible for iterating `response.aiter_lines()` and eventually awaiting `response.aclose()`") — but if a caller forgets, the connection stays open. This is an inherent httpx-stream gotcha, not a PR-introduced bug, but worth a logging-level connection-pool watchdog if not already present.
- **No tests for the actual stream resume end-to-end** against a fake SSE server — only schema-level transforms and proxy-ingress dispatch. A live-stream test would cost more setup but catch regressions in the whole pipeline.

## Suggestions

- **(Recommended)** Mirror the sync `try/except httpx.TimeoutException → litellm.Timeout` block in the async `get()` so behavior is symmetric.
- **(Recommended)** Add a negative test for `stream=false` ensuring the key is not added to `data`.
- **(Optional)** Add an end-to-end SSE stream-resume integration test using a fake httpx mock transport that emits a few SSE events starting from `sequence_number > starting_after`.
- **(Nit)** Type hint on `aget_responses` return is now `Union[ResponsesAPIResponse, BaseResponsesAPIStreamingIterator]` — consider a `@overload` pair so callers passing `stream=True` get the iterator type narrow without runtime introspection.

## Verdict: `merge-after-nits`

Correct fix at the right layer (proxy ingress) with thoughtful provider-transform plumbing. The sync/async asymmetry in exception handling is the only real concern; the missing negative test is a coverage gap, not a correctness gap.

## What I learned

The "GET request body is empty so kwargs from query string get silently dropped" bug class is one of those things that only surfaces when an SDK starts encoding optional params via the query string instead of headers. The right defense is at the framework layer: the proxy could enforce a "any GET handler that accepts `data: dict` must explicitly merge `request.query_params`" contract via a decorator, rather than relying on every handler author to remember. Worth considering as a follow-up.
