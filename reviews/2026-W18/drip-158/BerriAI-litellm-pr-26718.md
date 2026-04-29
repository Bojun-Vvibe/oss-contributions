# BerriAI/litellm#26718 — fix(http_handler): handle RequestNotRead in MaskedHTTPStatusError for multipart uploads

- PR: https://github.com/BerriAI/litellm/pull/26718
- Head SHA: `6f98bbe3323e5bd498f41419e7cc7a42f5c4b858`
- Author: dawidkulpa
- Diff: +31/-1 across `litellm/llms/custom_httpx/http_handler.py` and `tests/test_litellm/llms/custom_httpx/test_credential_leak_prevention.py`

## What changed

Defends `MaskedHTTPStatusError.__init__` against `httpx.RequestNotRead` when the original failed request was a streaming body (multipart/form-data, chunked uploads). At `http_handler.py:387-394` the previous code did `content=original_error.request.content` directly inside the `httpx.Request(...)` rebuild — which raises `httpx.RequestNotRead` for any request whose body is a `httpx.ByteStream`/streaming source that hasn't been buffered. The fix wraps the access in `try/except httpx.RequestNotRead`, falling back to `request_content = b""` and threading that into the rebuilt `masked_request`.

Test added at `test_credential_leak_prevention.py:107-130` (`test_handles_streaming_request_content`) builds an `httpx.Request("POST", "https://api.openai.com/v1/images/edits?key=SECRET_KEY", stream=httpx.ByteStream(b"multipart-data"))` plus a 400 response, wraps in `httpx.HTTPStatusError`, and asserts `MaskedHTTPStatusError(orig)` (1) doesn't crash, (2) preserves `status_code == 400`, (3) attaches the masked request, and (4) successfully scrubs `SECRET_KEY` from the URL.

## Observations

This is the right minimal fix for the reported regression class. `httpx.Request.content` is documented to raise `RequestNotRead` precisely when the body wasn't materialized, and the previous code path collapsed any masked-error construction for image edits, audio transcription, file uploads — exactly the surfaces where credentials in URLs are most likely to leak (image-edit endpoints often carry `?api_key=...` per provider quirks). Now those flows correctly produce a maskable error rather than crashing on the masking attempt.

The fallback `b""` is correct in this context because the `MaskedHTTPStatusError` exists to mask credentials in the request *URL/headers*, not to forensically preserve the request body. Dropping the body content is an acceptable security/observability tradeoff: a debug log inspecting `error.request.content` will see empty bytes rather than the original multipart payload, but the alternative — letting the original `RequestNotRead` propagate, hiding the upstream HTTP error entirely — is strictly worse. A one-line comment to that effect (`# body is intentionally dropped: streaming bodies cannot be safely cloned post-error, and the masking contract only covers URL/headers`) at the new `try/except` site would help the next reader.

The test asserts `"SECRET_KEY" not in str(masked.request.url)` which exercises the masking path in addition to the no-crash invariant — that's the load-bearing combination. One missed assertion: confirming `masked.request.content == b""` (not the original `b"multipart-data"`) would pin the fallback shape and prevent a future "preserve body" refactor from silently re-introducing the leak vector that motivated `MaskedHTTPStatusError` in the first place.

The headers dict comprehension at line 384 (`if k.lower() not in ("content-encoding", "content-length")`) is preserved unchanged — appropriate, since dropping `content-length` is now even more important given the body is empty: a stale `Content-Length: <original>` header on a 0-byte body would mislead any downstream that re-derives the request shape.

No banned content; no secret introduced (the test uses literal `SECRET_KEY`/`KEY_X` placeholders that already exist in this file's other cases).

## Verdict

`merge-after-nits`
