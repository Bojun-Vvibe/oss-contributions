# BerriAI/litellm #26633 — fix: handle image edit mask multipart errors without RequestNotRead

- URL: https://github.com/BerriAI/litellm/pull/26633
- Head SHA: `a64416b6fda0cd6736026e0e7b97cc6e60a418f5`
- Diff: +112/-9 across `litellm/llms/custom_httpx/http_handler.py` and three test files. Closes #26552.

## Context / problem

When an image-edit request built with `httpx.Request(..., data=..., files=[...])` fails upstream and `MaskedHTTPStatusError.__init__` tries to rebuild the request with the secret URL masked, the constructor accesses `original_error.request.content`. For multipart streaming bodies (`image[]` + `mask` BytesIO streams) httpx hasn't materialized the body yet and raises `httpx.RequestNotRead: Attempted to access streaming request content, without having called read()`. The masking helper itself crashes, the original `HTTPStatusError` (with the unmasked URL containing the API key in the query string) propagates up the stack, and the user sees a stack trace with the secret in plain text — both a crash *and* a credential-leak.

## What the fix does

`http_handler.py:348-365` extracts a new `_build_masked_request(original_request, masked_url) -> httpx.Request` helper:

```python
def _build_masked_request(original_request, masked_url):
    headers = httpx.Headers(original_request.headers)
    try:
        request_content = original_request.content
    except httpx.RequestNotRead:
        request_content = b""
        headers.pop("content-length", None)
        headers.pop("transfer-encoding", None)
    return httpx.Request(method=..., url=masked_url, headers=headers, content=request_content)
```

The `RequestNotRead` catch substitutes an empty body and — critically — strips `content-length` and `transfer-encoding` so the rebuilt `httpx.Request` doesn't claim to carry a body it doesn't have (which would itself raise on later inspection). Then `MaskedHTTPStatusError.__init__` (`:410`) replaces its inline request reconstruction with `_build_masked_request(original_error.request, masked_url)`.

## Specific references

- `http_handler.py:348-365` — `_build_masked_request`. The `headers.pop("content-length", None)` and `headers.pop("transfer-encoding", None)` after substituting `b""` are the load-bearing pieces. Without them, downstream code that re-derives body length from headers (or tries to read `b""` as a streaming body of length N) would crash again.
- `http_handler.py:410` — call-site swap. The pre-fix code passed `original_error.request.headers` directly into the new `httpx.Request`, which (a) doesn't strip `content-encoding`/`content-length` and (b) accesses `.content` unconditionally. Both holes are closed by the helper.
- `tests/test_litellm/llms/custom_httpx/test_credential_leak_prevention.py:139-159` — `test_handles_unread_multipart_request_content`. Builds an actual `httpx.Request` with `files=[("image[]", ("image.png", BytesIO(b"image"), "image/png")), ("mask", ("mask.png", BytesIO(b"mask"), "image/png"))]`, wraps it in a 500 response, then asserts `MaskedHTTPStatusError(orig)` succeeds, `"SECRET" not in str(masked.request.url)`, `masked.request.content == b""`, and `masked.request.headers.get("content-length") == "0"` — pins the no-crash, the URL mask, the body-substitution, and the header reconciliation in one test.
- `tests/test_litellm/llms/openai/test_openai_image_edit_transformation.py:267-300` — `test_transform_image_edit_request_with_bytesio_mask_list`. Adds parallel coverage for the proxy-style BytesIO mask-list path on the request-build side (asserts the multipart files list contains the right `("mask", ("mask.png", mask, ...))` tuple). This is what reproduces the conditions that lead to the unread-stream state.
- `tests/test_litellm/proxy/image_endpoints/test_azure_routes.py` — adds `test_azure_image_edit_route` extension to round-trip the same shape through the Azure adapter routing.

## Risks / nits

- `request_content = b""` substitutes silently — if a caller is relying on `masked.request.content` to debug what was sent, they'll see empty bytes for the multipart-streaming case. Worth a one-line code comment naming this trade-off ("we can't safely re-read the streaming body without consuming it; mask body shape rather than content"). The PR body explains this but the inline code comment doesn't.
- The `RequestNotRead` catch is the only `httpx`-specific exception handled; if httpx grows more "body not yet materialized" exception types (or if the stream is partially read and raises `StreamConsumed`), they'd surface as the same crash. Cheapest defense is `except (httpx.RequestNotRead, httpx.StreamConsumed)` or a broader `except Exception` with a structured warn-log.
- Pre-Submission checklist `make test-unit` checkbox is unchecked. Maintainers should confirm CI actually ran the new credential-leak test suite.

## Verdict

`merge-as-is` — surgical fix at the exact crash site, with the right header-reconciliation step (strip `content-length` + `transfer-encoding`) so the substituted empty body doesn't trigger a second crash downstream, and three layers of test coverage (the masking helper itself, the request transformation that creates the unread-stream condition, and the proxy route end-to-end). The unchecked `make test-unit` box and the missing inline trade-off comment are cosmetic.
