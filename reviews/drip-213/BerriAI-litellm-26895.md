# Review: BerriAI/litellm#26895 — fix: preserve aiohttp raw response headers

- PR: https://github.com/BerriAI/litellm/pull/26895
- Head SHA: `8870a0f9c5c0eda738ed16737aba441b6de7ef38`
- Closes: #26548
- Files: `litellm/llms/custom_httpx/aiohttp_transport.py` (+1/-1), `tests/test_litellm/test_streaming_connection_cleanup.py` (+47/-0)
- Verdict: **merge-as-is**

## What it does

One-character substantive fix at `aiohttp_transport.py:344`: when constructing the
`httpx.Response` from an aiohttp response, pass `getattr(response, "raw_headers", response.headers)`
instead of `response.headers`. The `raw_headers` attribute is a sequence of
`(bytes, bytes)` tuples that aiohttp populates with the wire-form headers; httpx's
`Response` constructor decodes those bytes itself (using its own header-decoding logic),
which preserves non-ASCII (UTF-8) header values. The previous `response.headers` path
went through aiohttp's already-decoded `CIMultiDict[str]` which assumes ISO-8859-1 for
header values and mangles UTF-8.

## Notable changes

- `aiohttp_transport.py:344` — the singular line change. `getattr(...)` with a default
  preserves backward compatibility for any fake/mock that doesn't expose `raw_headers`
  (existing tests use such mocks).
- `tests/.../test_streaming_connection_cleanup.py:67-114` — new
  `test_aiohttp_transport_preserves_non_ascii_response_headers` regression test that:
  - Defines `header_value = "视频生成成功"` (the exact failure-mode payload from the linked
    issue, a Volcano Ark / ByteDance video-gen response header).
  - Builds a `FakeSession` whose response object exposes both `headers` (decoded dict
    with the original Chinese string) and `raw_headers` (tuple-of-bytes form with the
    UTF-8 encoding). This is exactly the surface aiohttp presents at runtime.
  - Calls `transport.handle_async_request(...)` and asserts the round-trip:
    `response.headers["x-ark-message"] == header_value`. Note the test uses lower-case
    key lookup, which exercises httpx's case-insensitive header matching.
  - Asserts `await response.aread() == b"ok"` to confirm the body stream still works.

## Reasoning

Bytes-vs-decoded-string at HTTP transport boundaries is a known footgun: HTTP headers
are technically ISO-8859-1 per RFC 7230, but in practice many servers emit UTF-8 in
custom headers. aiohttp's `headers` property pre-decodes; httpx accepts raw bytes and
decodes itself. Routing the bytes through to the destination decoder is the correct
shape — the alternative ("re-encode aiohttp's already-decoded string back to bytes") is
lossy when the original encoding was anything other than what aiohttp guessed.

The `getattr(..., default=response.headers)` form is the right minimal-blast-radius
approach: real aiohttp always has `raw_headers`, fakes / older mocks might not, and the
default falls back to the prior behaviour. Existing tests that don't define `raw_headers`
on their fake response objects are unaffected.

Test coverage is tight — the chosen header value is from the linked issue, the fake's
shape mirrors real aiohttp's dual surface, and the assertion uses the same case-insensitive
lookup pattern that broke for the reporter.

## Nits

None worth blocking on. Optional follow-ups:
- Could add a second test where `raw_headers` is absent (dict-only fake) to pin the
  fallback branch — the PR body claims "keep the existing decoded-header fallback for
  test/fake responses that do not expose raw_headers" but no test actively exercises
  the fallback. If the existing `test_aiohttp_transport_creates_response` test already
  uses a `raw_headers`-less fake then that is implicit coverage.
- The test file is named `test_streaming_connection_cleanup.py` but this test is about
  header decoding, not streaming. Functionally fine (the file is the home for aiohttp
  transport tests), just a small naming smell.
