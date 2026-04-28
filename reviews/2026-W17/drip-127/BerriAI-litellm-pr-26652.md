# BerriAI/litellm #26652 — fix(aiohttp): sanitize non-ASCII response headers before passing to httpx

- URL: https://github.com/BerriAI/litellm/pull/26652
- Head SHA: `d1cef71d48b0e9f09b01ce34200f6a2bd200f1f6`
- Verdict: **merge-as-is**

## Review

- Real bug fix at `litellm/llms/custom_httpx/aiohttp_transport.py:342-360` for issue #26548 — Volcengine Ark and similar providers return response headers containing non-ASCII characters (Chinese text in `x-tos-expiration`'s `rule-id="180day自动清理"`, or `x-ark-message: "视频生成成功"`). aiohttp decodes those into Python `str` happily, but `httpx.Headers.__init__` calls `value.encode("ascii")` on every value and raises `UnicodeEncodeError` — which crashes the *entire* request including the body the caller actually wanted. Pre-fix, any model call returning a media asset URL with one of these headers was unrecoverable.
- The sanitize loop at `:351-360` is the load-bearing piece: try `value.encode("ascii")` per header, on success append `(key, value)` (str), on `UnicodeEncodeError` append `(key, value.encode("utf-8"))` (bytes). The choice to pass UTF-8 bytes rather than e.g. drop the header or replace non-ASCII with `?` is correct — httpx accepts a list-of-tuples where values can be `str | bytes`, and storing bytes bypasses httpx's str→ascii encoding step entirely. Headers are preserved end-to-end and the request completes.
- The comment at `:342-350` is exactly the kind of inline documentation that prevents the bug from being reintroduced. It names: (a) the upstream cause (aiohttp's permissive str decode), (b) why httpx blows up (`value.encode("ascii")` strict path), (c) why bytes is the workaround (httpx stores them as-is), and (d) the user-visible consequence (later access via `response.headers[key]` will use latin-1 decode so multi-byte UTF-8 may appear garbled but the *request completes*). The "garbled but works" tradeoff is the right call — these headers are not load-bearing for callers; they're metadata that may not even be read.
- The `typing.List[typing.Any]` annotation at `:353` plus `# type: ignore[arg-type]` at `:363` is a small lint smell. The actual type is `List[Tuple[str, Union[str, bytes]]]` and `httpx.Response(headers=...)` accepts that. Tightening the annotation would let mypy verify the contract. Not a blocker.
- Test at `tests/test_litellm/llms/custom_httpx/test_aiohttp_transport.py:564-635` is the right shape and covers all the invariants:
  - Asserts the request *doesn't crash* (line `:611` "This should NOT raise UnicodeEncodeError").
  - Asserts ASCII-safe headers (`content-type`) are preserved as `str` (the type-preserving fast path).
  - Asserts non-ASCII headers are *present* in the response (not silently dropped).
  - Asserts ASCII *portions* of mixed headers (`"180day"`, `"rule-id"`) are preserved through the bytes-encoding round-trip — important because if the implementation accidentally re-encoded the whole header it could lose the structure.
  - Asserts raw UTF-8 bytes are present (`"自动清理".encode("utf-8") in raw_headers[b"x-tos-expiration"]`) — pins the contract that the original encoding survives, which is the recoverable path for any caller that wants to manually decode.
- Realistic test fixtures (Volcengine's actual header names and Chinese phrasings) make the test self-documenting. The `MockSession` shape matches aiohttp's contract closely enough that future aiohttp version bumps that change the response interface will fail this test loudly.
- Edge case worth thinking about for follow-up but not blocking: what about *header keys* that contain non-ASCII? RFC 7230 says they shouldn't, but the same providers that emit non-ASCII values have been known to emit non-ASCII keys too. Current code only sanitizes values; non-ASCII keys would still throw. Probably out of scope for this PR.
