# BerriAI/litellm PR #26598 — preserve non-ASCII response headers through the aiohttp transport

- **PR**: https://github.com/BerriAI/litellm/pull/26598
- **Author**: @zyz23333
- **Head SHA**: `45336b0179c7e24505c9035ebb0b70962dc70a82`
- **Size**: +74 / −6
- **Files**: `litellm/llms/custom_httpx/aiohttp_transport.py`, `litellm/llms/custom_httpx/http_handler.py`, `tests/test_litellm/llms/custom_httpx/test_aiohttp_transport.py`, `tests/test_litellm/llms/custom_httpx/test_credential_leak_prevention.py`, `tests/test_litellm/test_streaming_connection_cleanup.py`

## Summary

Two-line behavior fix: the aiohttp→httpx bridge was passing `response.headers` (a string-decoded `CIMultiDict` that aiohttp will refuse to construct from non-ASCII bytes) into `httpx.Response`, and `MaskedHTTPStatusError` was iterating `original_error.response.headers.items()` and decoding to strings — both lose non-ASCII header values. The fix passes `response.raw_headers` (bytes-tuples) and switches the masking helper to iterate `response.headers.raw` so byte-fidelity round-trips end-to-end.

## Verdict: `merge-after-nits`

Correct bug, correct fix, real upstream test coverage including the "credential redaction still works on byte-level headers" path. The nit is that `aiohttp_transport.py` now hands httpx a value (`raw_headers`) whose shape isn't immediately obvious to future readers, and the four test fixtures grow a `raw_headers = ()` field in lockstep — a subtle invariant that would benefit from a one-line comment in the transport.

## Specific references

- `litellm/llms/custom_httpx/aiohttp_transport.py:341-345` — the entire fix is `headers=response.raw_headers` instead of `headers=response.headers`. `aiohttp.ClientResponse.raw_headers` is `Tuple[Tuple[bytes, bytes], ...]`, which `httpx.Response` accepts directly via its overloaded `headers` constructor and which preserves the original UTF-8 / latin-1 bytes the upstream sent. The previous code decoded via aiohttp's `CIMultiDictProxy[str]`, which raises on non-ASCII *or* lossily decodes depending on aiohttp version — an upstream provider returning a localized error message in `X-RateLimit-Reason` (or any non-ASCII header) would corrupt or 500.
- `litellm/llms/custom_httpx/http_handler.py:381-389` — `MaskedHTTPStatusError` now builds `response_headers` as `[(k, v) for k, v in original_error.response.headers.raw if k.lower() not in (b"content-encoding", b"content-length")]`. The `.lower()` on bytes works because ASCII-lowercase is byte-stable for the two header names being filtered. Good. The list-of-tuples shape is what `httpx.Response(headers=...)` accepts.
- `tests/test_litellm/llms/custom_httpx/test_credential_leak_prevention.py:138-155` — `test_preserves_non_ascii_response_headers` constructs a Chinese-text header value, wraps a redacted-URL request in `MaskedHTTPStatusError`, and asserts both that the header round-trips intact (`response.headers.get("x-localized-message") == header_value`) AND that the URL secret is still redacted (`"SECRET" not in str(masked.response.request.url)`). This is the right test shape — it pins the bug fix and the credential-redaction invariant in the same assertion.
- `tests/test_litellm/llms/custom_httpx/test_aiohttp_transport.py:265-308` — `test_handle_async_request_preserves_non_ascii_response_headers` builds a `FakeSession.Resp` with both `headers={"X-Localized-Message": "本地化消息"}` AND `raw_headers=((b"X-Localized-Message", "本地化消息".encode("utf-8")),)`. The dual surface mirrors the live aiohttp shape and guarantees the transport pulls from `raw_headers`, not the decoded dict.
- The `raw_headers = ()` additions to four pre-existing FakeSession test fixtures (test_aiohttp_transport.py:241, 344, 403, plus test_streaming_connection_cleanup.py:43, 76) are mechanical — they prevent `AttributeError: 'Resp' object has no attribute 'raw_headers'` after the transport switch.

## Nits

1. Add a one-line comment at `aiohttp_transport.py:343` — `# raw_headers preserves bytes; response.headers is str-decoded and rejects non-ASCII` — so the next reader doesn't "fix" this back.
2. The `(b"content-encoding", b"content-length")` tuple in `http_handler.py:386` is slightly fragile; if a future header name ever needs case-folding beyond ASCII, this needs revisiting. Today it's fine because both names are pure-ASCII.
3. There's a parallel PR (#26593) doing the same fix as a one-liner with no test coverage of the masking path. This PR supersedes it; the maintainers should close #26593 with a pointer here.

## What I learned

aiohttp and httpx disagree on header type fidelity: aiohttp gives you both `headers: CIMultiDictProxy[str]` (decoded, may raise) and `raw_headers: Tuple[Tuple[bytes, bytes], ...]` (untransformed). When bridging the two, always pass `raw_headers` — the cost is a tuple-of-tuples allocation, the win is byte-fidelity through provider responses that include localized error text or B2B trace IDs. The credential-leak test in `test_preserves_non_ascii_response_headers` is also a nice pattern for "fix the bug AND assert the existing security invariant in the same test" — it would have caught a regression that fixed the encoding issue by accidentally bypassing redaction.
