---
pr: 26550
repo: BerriAI/litellm
sha: e29123318e185beb788a9d15e9cd57ed7d0ebcca
verdict: merge-after-nits
date: 2026-04-27
---

# BerriAI/litellm #26550 — fix(custom_httpx): preserve non-ascii response headers

- **Head SHA**: `e29123318e185beb788a9d15e9cd57ed7d0ebcca`
- **Author**: zyz23333 (Yancey Zhou)
- **Size**: small (2 source hunks in `aiohttp_transport.py` + `http_handler.py` + 2 test files) — bundled with same unrelated formatter-only diffs as #26551

## Summary
`aiohttp` exposes parsed (decoded) headers as a `multidict[str, str]` via `.headers` and the raw bytes-tuple form via `.raw_headers` / `.headers.raw`. When upstream APIs return non-ASCII header values (e.g. localized error messages, RFC-2047 mojibake from misconfigured CDNs), aiohttp's parsed form has already done a lossy decode. Switching to the raw-bytes form lets httpx do its own (lossless) decoding downstream.

## Specific findings
- `litellm/llms/custom_httpx/aiohttp_transport.py:343-345` — one-line change: `headers=response.headers` → `headers=response.raw_headers`. `httpx.Response` accepts an iterable of `(bytes, bytes)` tuples for the `headers` arg, so passing `aiohttp`'s `raw_headers` (which is exactly that shape per aiohttp docs) is the correct fix. The previous `response.headers` was a `CIMultiDict[str, str]` which httpx accepted but had already been latin-1 (or replacement-char) decoded by aiohttp.
- `litellm/llms/custom_httpx/http_handler.py:381-388` — parallel fix in the credential-leak masker: was iterating `original_error.response.headers.items()` and filtering out `content-encoding` / `content-length` as `str`. Now iterates `original_error.response.headers.raw` and compares as `bytes` (`b"content-encoding"`, `b"content-length"`). Correct and symmetric. Edge case: `headers.raw` is httpx-specific (this is the *masker* path so it operates on `httpx.Response`); `aiohttp_transport` uses `response.raw_headers` because *that* path operates on an `aiohttp.ClientResponse`. Two different APIs, both correctly named. Worth a one-line comment in each file explaining which client library it's reading from — easy to confuse on later reading.
- `tests/test_litellm/llms/custom_httpx/test_aiohttp_transport.py:240-345` — three existing test fixtures get `raw_headers = ()` added so they don't break against the new code path. Good defensive update.
- `tests/test_litellm/llms/custom_httpx/test_aiohttp_transport.py:266-310` — new `test_handle_async_request_preserves_non_ascii_response_headers` uses Chinese characters (`本地化消息`) for the header value and asserts `response.headers.get("x-localized-message") == header_value`. Solid regression test. One concern: `httpx.Response`'s default decoding for header bytes is also strict — confirm the assertion passes locally on the head SHA, because if httpx silently latin-1-decodes UTF-8 bytes you'd get mojibake, not the original string. The PR author should run this test against the actual httpx version pinned in `pyproject.toml`.
- `tests/test_litellm/llms/custom_httpx/test_credential_leak_prevention.py:135-158` — the existing `test_strips_content_encoding_to_avoid_double_decode` is updated (per the diff context) but the diff doesn't show a parallel non-ASCII test for the masker path. Add one — the `http_handler.py:381` change is the more security-relevant of the two (logs leaking decoded vs raw bytes), so it deserves its own regression.
- Bundled unrelated formatting diffs: same `factory.py`, `predibase/transformation.py`, `xecguard.py` reformats that #26551 also bundles. Split them out.

## Risk
Low for the fix; medium-low for "downstream consumers of `httpx.Response.headers` may have been silently relying on aiohttp's lossy decode to coerce binary garbage to printable ASCII". Worth a quick search for `response.headers[` access patterns that don't tolerate non-ASCII.

## Verdict
**merge-after-nits** — add a parallel non-ASCII test for the `http_handler.py` masker path, drop one comment in each file naming the client library being read, and unbundle the formatter diffs.
