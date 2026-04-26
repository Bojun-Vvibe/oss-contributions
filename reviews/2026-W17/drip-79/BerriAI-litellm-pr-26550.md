# BerriAI/litellm PR #26550 — fix(custom_httpx): preserve non-ascii response headers

- **PR:** https://github.com/BerriAI/litellm/pull/26550
- **Author:** zyz23333 (Yancey Zhou)
- **Head SHA:** `21d98780cc0525246a38ad701a0471a611ebb2e5`
- **Files:** 5 (+74 / -6)
- **Verdict:** `merge-as-is`

## What it does

Fixes #26548. When the `LiteLLMAiohttpTransport` bridged an aiohttp response into an `httpx.Response`, it passed `response.headers` (a `multidict.CIMultiDict[str, str]`) directly. `httpx.Headers` then attempted to ASCII-encode any non-ASCII string values, raising:

```
UnicodeEncodeError: 'ascii' codec can't encode characters in position 0-5: ordinal not in range(128)
```

The fix uses `response.raw_headers` (a tuple of `(bytes, bytes)`) instead, which `httpx.Headers` accepts as already-encoded UTF-8 bytes. Same fix is applied in the credential-leak masked-error path.

## Specific reads

- `litellm/llms/custom_httpx/aiohttp_transport.py:344` — single-line core fix:
  ```python
  return httpx.Response(
      status_code=response.status,
-     headers=response.headers,
+     headers=response.raw_headers,
      stream=AiohttpResponseStream(response),
      request=request,
  )
  ```
  `aiohttp.ClientResponse.raw_headers` returns `Tuple[Tuple[bytes, bytes], ...]` (header name + value as raw bytes off the wire). `httpx.Headers.__init__` accepts an iterable of `(bytes, bytes)` pairs and stores them without re-encoding — this is exactly the right shim.
- `litellm/llms/custom_httpx/http_handler.py:381-388` — masked-error path swap:
  ```python
- response_headers = {
-     k: v
-     for k, v in original_error.response.headers.items()
-     if k.lower() not in ("content-encoding", "content-length")
- }
+ response_headers = [
+     (k, v)
+     for k, v in original_error.response.headers.raw
+     if k.lower() not in (b"content-encoding", b"content-length")
+ ]
  ```
  Type-shape change `dict[str, str]` → `list[tuple[bytes, bytes]]`. Confirm the downstream `httpx.Response(..., headers=response_headers, ...)` constructor accepts the list form (it does — same constructor as the transport path). The `b"content-encoding"` literal is correct since `original_error.response.headers.raw` yields bytes pairs.
- `litellm/llms/custom_httpx/http_handler.py:387` — `b"content-length"` filter is preserved in bytes form. Important: dropping these is critical to avoid double-decoding when the body is rebuilt; the bytes-vs-str predicate change is the only delta.
- `tests/test_litellm/llms/custom_httpx/test_aiohttp_transport.py:266-308` — new test `test_handle_async_request_preserves_non_ascii_response_headers` constructs a mock with `raw_headers = ((b"X-Localized-Message", header_value.encode("utf-8")),)` and asserts the round-tripped `response.headers.get("x-localized-message") == header_value`. Tight regression test, decodes back to the original `本地化消息` string. Good.
- `tests/test_litellm/llms/custom_httpx/test_credential_leak_prevention.py:138-153` — companion test for `MaskedHTTPStatusError` builds an `httpx.Response` directly with `headers=[(b"x-localized-message", header_value.encode("utf-8"))]` and confirms (a) the masked response preserves the non-ASCII header, (b) the `SECRET` query string is still scrubbed from the URL. Good — checks both the bug fix and that the existing credential-masking contract isn't regressed.
- `test_aiohttp_transport.py:241,344,403` and `test_streaming_connection_cleanup.py:43,77` — five existing test doubles get `raw_headers = ()` added to their `Resp`/`MockResponse` mocks. Necessary because the production code now reads `raw_headers` on every code path; without these the unrelated tests would `AttributeError`. Mechanical, correct.

## Risk surface

- **Type-shape change in masked-error path**: `dict` → `list[tuple]` is accepted by `httpx.Response`, but anything downstream that re-reads `response_headers` as a dict via `.items()` would break. `grep -n response_headers` in the immediate file suggests it's only consumed by the constructor.
- **`raw_headers` availability**: `aiohttp.ClientResponse.raw_headers` has been stable since aiohttp 3.x. The existing minimum aiohttp constraint in `pyproject.toml` should already satisfy this — worth a 5-second check but not a blocker.
- **Test mocks**: the five `raw_headers = ()` additions silently coerce to "no headers" in the mocks, which is fine for the existing assertions (none of them inspect headers).

Verdict: merge as-is. Surgical bug fix on a real `UnicodeEncodeError` path, two clean regression tests covering both the transport and the credential-masking sites, and the test-mock updates are mechanically required by the production change. No spurious churn.
