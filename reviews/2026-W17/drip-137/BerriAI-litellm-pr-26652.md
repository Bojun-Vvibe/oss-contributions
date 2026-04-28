# BerriAI/litellm #26652 — fix(aiohttp): sanitize non-ASCII response headers before passing to httpx

- PR: https://github.com/BerriAI/litellm/pull/26652
- Head SHA: `d1cef71d48b0`
- Diff: 96+/1- across 2 files (`litellm/llms/custom_httpx/aiohttp_transport.py`, `tests/test_litellm/llms/custom_httpx/test_aiohttp_transport.py`)
- Base: `litellm_internal_staging` · Fixes #26548

## Verdict: merge-as-is

## Rationale

- **Closes a real `UnicodeEncodeError` crash with a precisely-named upstream trigger.** Pre-diff `aiohttp_transport.py:343` passed `response.headers` (an `aiohttp` headers dict that may contain Python `str` with non-ASCII codepoints) directly into `httpx.Response(... headers=response.headers)`. `httpx.Headers.__init__` normalizes each value via `value.encode("ascii")` which raises `UnicodeEncodeError` on the first multi-byte codepoint — bringing down the entire request. The PR body names the canonical reproducer (Volcengine Ark's `x-tos-expiration: rule-id="180day自动清理"`), and the test cell adds a second known-affected header `x-ark-message: 视频生成成功`.
- **The sanitization loop at `:342-358` is correctly minimal.** Walks `response.headers.items()`, attempts `value.encode("ascii")` (no allocation kept — just a probe), and on success appends the original `(key, value)` pair to `sanitized_headers`. On `UnicodeEncodeError` it falls back to `(key, value.encode("utf-8"))` — passing bytes directly to `httpx.Response` which stores them as-is and *skips the str→ascii encoding step*. ASCII-safe headers (the common case for content-type, content-length, etc.) take the fast path with zero allocation overhead.
- **The trade-off documented in the inline comment at `:339-352` is the right one and load-bearing.** "When later accessed via response.headers[key], httpx decodes the stored bytes (typically with latin-1), so multi-byte UTF-8 sequences may appear garbled — but the request completes successfully instead of crashing." That's the canonical "fail-soft instead of crashing on a header you probably don't read anyway" trade. The Volcengine `x-tos-expiration` header carrying the Chinese `180day自动清理` is a TOS expiration metadata field that downstream litellm doesn't consume — getting garbled latin-1 representation is strictly better than a 500.
- **Test coverage is the right shape — five assertions cover the full contract.** At `test_aiohttp_transport.py:561-635`:
  - `response.status_code == 200` (request didn't crash)
  - `response.headers["content-type"] == "application/octet-stream"` (ASCII-safe header preserved as-is, no regression)
  - `"x-tos-expiration" in response.headers` (non-ASCII header is still present)
  - `"180day" in tos_value` and `"rule-id" in tos_value` (ASCII portions survive the bytes round-trip)
  - **Most importantly**, `"自动清理".encode("utf-8") in raw_headers[b"x-tos-expiration"]` — pinning that the *original UTF-8 bytes are preserved in the raw representation*, so any consumer that bypasses `response.headers[key]` and reads `response.headers.raw` gets the recoverable original bytes
  This is the correct contract: the typed-`str` access path is best-effort latin-1-decoded, the raw-bytes path is faithful.
- **`type: ignore[arg-type]` at `:362` is correctly placed and necessary.** `httpx.Response`'s `headers` parameter is typed `HeaderTypes` which is a `Sequence[Tuple[str, str]]`-ish union that doesn't include `Sequence[Tuple[str, bytes]]`, but httpx accepts the bytes shape at runtime. The `type: ignore` with the specific code (`arg-type` not bare) is the right precision.

## Nits / follow-ups

- **The `try/except UnicodeEncodeError` per header is O(n) attempts** for the headers-mostly-ASCII case. For a typical response with 10-15 headers this is fine, but for an LLM-streaming response with many `x-` provider-specific headers and a hot inner loop, the per-header try/except + per-header tuple append is ~3× slower than `httpx.Headers(response.headers, encoding="utf-8")` *if* httpx supported a non-ASCII default encoding (it doesn't, which is why this workaround exists). Worth a perf microbenchmark if request rates are high — but for the failure-mode-being-fixed, correctness > perf.
- **No test cell for the case where the header *value* is already bytes** (e.g. some lower-level aiohttp paths set raw bytes). The current code does `value.encode("ascii")` which would crash with `AttributeError: 'bytes' object has no attribute 'encode'` if value is already bytes. Worth a probe — `if isinstance(value, bytes): sanitized_headers.append((key, value)); continue` at the top of the loop. Probably unreachable given aiohttp's typed dict, but defensive.
- **The header *key* is not checked.** If the upstream returns a non-ASCII header *name* (vanishingly rare per RFC 7230 §3.2 which limits names to token chars, but malicious servers could try), the code would still pass the str key to httpx which would raise `UnicodeEncodeError` on the key encoding too. Worth either symmetrically sanitizing the key or asserting at the loop top that the key is ASCII (fail-loud rather than fail-on-httpx-internals).
- **The `value.encode("ascii")` probe allocates a bytes object that's immediately discarded.** A faster probe would be `value.isascii()` (Python 3.7+) — same semantics, no allocation. Worth swapping for `if value.isascii(): sanitized_headers.append((key, value)) else: sanitized_headers.append((key, value.encode("utf-8")))`.

## What I learned

The httpx `Headers` constructor's hard-coded `value.encode("ascii")` normalization is a known sharp edge for any provider that returns localized header values — RFC 7230 §3.2.4 technically forbids non-ASCII in headers but real-world providers (Volcengine, some Alibaba Cloud services, internal Chinese APIs) routinely violate it. The "encode as UTF-8 bytes, pass through, accept latin-1-decoded read-back" pattern is the canonical workaround. The `value.isascii()` probe vs `value.encode("ascii")` try/except is the perf-conscious form. The most important test-cell shape is the *raw bytes round-trip assertion* (`"自动清理".encode("utf-8") in raw_headers[...]`) — without that, a future "fix" that drops to ASCII-only with `?` substitution would still pass the typed-str-access tests but lose information.
