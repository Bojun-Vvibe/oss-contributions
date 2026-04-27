# BerriAI/litellm#26550 — fix(custom_httpx): preserve non-ascii response headers

- **Head**: `e29123318e185beb788a9d15e9cd57ed7d0ebcca`
- **Size**: +93/-13 across 8 files
- **Verdict**: `merge-after-nits`

## Context

Fixes #26548. When the aiohttp transport in `LiteLLMAiohttpTransport` received a response header with a non-ASCII value (e.g. a localized error message from an upstream provider), bridging into `httpx.Response(...)` raised `UnicodeEncodeError: 'ascii' codec can't encode characters in position 0-5: ordinal not in range(128)`. Root cause: passing `aiohttp.ClientResponse.headers` (already decoded to `str` via aiohttp's mojibake-prone path) into `httpx.Headers`, which then attempted to ASCII-normalize it.

## Design analysis

### Primary fix: `litellm/llms/custom_httpx/aiohttp_transport.py:344`

```python
return httpx.Response(
    status_code=response.status,
-    headers=response.headers,
+    headers=response.raw_headers,
    stream=AiohttpResponseStream(response),
    request=request,
)
```

`aiohttp.ClientResponse.raw_headers` is `Tuple[Tuple[bytes, bytes], ...]` — exactly what `httpx.Response` accepts as a sequence-of-tuples-of-bytes form, bypassing the str round-trip that triggered the ASCII normalize. This is the surgical right fix: one identifier change, semantically clean (raw byte preservation through the bridge), zero contract change for downstream callers (httpx normalizes raw bytes into its own `Headers` type which handles non-ASCII via latin-1).

### Mirror fix: `litellm/llms/custom_httpx/http_handler.py:381-388`

The masked-error path that rebuilds an `httpx.Response` for `MaskedHTTPStatusError` had the same str-source issue:

```python
-response_headers = {
-    k: v
-    for k, v in original_error.response.headers.items()
-    if k.lower() not in ("content-encoding", "content-length")
-}
+response_headers = [
+    (k, v)
+    for k, v in original_error.response.headers.raw
+    if k.lower() not in (b"content-encoding", b"content-length")
+]
```

Two correct semantic deltas: (1) iterates over `.headers.raw` (bytes-bytes pairs) instead of `.items()` (str-str), (2) the filter comparison correctly switches `("content-encoding", "content-length")` → `(b"content-encoding", b"content-length")` to match the byte-typed key. The shape change from dict-comprehension to list-comprehension is also correct — `httpx.Response(headers=…)` accepts both, but a list-of-tuples preserves duplicate header names (e.g. `Set-Cookie`), which a dict would silently dedupe.

### Test coverage

`tests/test_litellm/llms/custom_httpx/test_aiohttp_transport.py:266-309` adds `test_handle_async_request_preserves_non_ascii_response_headers` — constructs a fake `aiohttp.ClientResponse` with `raw_headers = ((b"X-Localized-Message", "本地化消息".encode("utf-8")),)`, runs through `LiteLLMAiohttpTransport.handle_async_request`, asserts `response.headers.get("x-localized-message") == "本地化消息"`. That's the right pin for the primary fix.

Three existing test doubles at `:241`, `:344`, `:403` were updated to add `raw_headers = ()` so they continue to satisfy the new `.raw_headers` access pattern. Correct.

The credential-leak test file at `tests/test_litellm/llms/custom_httpx/test_credential_leak_prevention.py:135-158` (per the diff hunk) gets a new assertion exercising the masked-error path's non-ASCII branch.

## Concerns / nits

1. **Drive-by formatter churn.** The PR includes:
   - `litellm/litellm_core_utils/prompt_templates/factory.py:5291-5295` — a `.format(response_schema)` reflow with no semantic change
   - `litellm/llms/predibase/chat/transformation.py:175-179, 209-211, 350-352` — three black-formatter line-wrap changes
   - `litellm/proxy/guardrails/guardrail_hooks/xecguard/xecguard.py:212-220, 588-589` — line-wrap and missing-EOF-newline

   None of these touch the bug-fix logic. They're consistent with running `make format` on the whole repo, but they bloat the PR diff and make `git blame` noisier than it needs to be. Cleaner to either split formatter churn into a separate commit or scope `make format` to the actually-touched files.
2. **Missing CI links in PR template.** All three "CI status guideline" links (`Branch creation CI run`, `CI run for the last commit`, `Merge / cherry-pick CI run`) are empty. Project's required-checklist pattern wants these populated.
3. **`raw_headers` semantics depend on aiohttp version.** `ClientResponse.raw_headers` has been stable since aiohttp 3.0 (2018), so this isn't a real concern in practice, but if the project's `requirements.txt` floor is older than that the test fixture's `raw_headers = ()` could mask incompatibility. Worth a quick `aiohttp>=3.0` confirmation in the relevant requirements file.
4. **Non-UTF-8 raw header bytes.** The test uses `header_value.encode("utf-8")` which assumes the upstream encodes UTF-8. RFC 9110 actually permits ISO-8859-1 with extension to other encodings via RFC 8187 — if the upstream returns latin-1 bytes for the header value, `httpx.Headers` will decode them as latin-1, which may produce mojibake. Not a regression vs the old code (which would have raised `UnicodeEncodeError` in *both* cases); just worth a docstring note that "raw bytes are passed through; encoding is httpx's call" so future readers don't mis-attribute mojibake bugs to this PR.

## Verdict reasoning

The core fix is two surgical identifier swaps (`.headers` → `.raw_headers`, `.items()` → `.raw` with byte-typed filter), each with a documented test pin. The mirroring across both the success and masked-error paths is correct and complete. The drive-by formatter churn and unfilled CI checklist drop the verdict from `merge-as-is` to `merge-after-nits`; the bug fix itself is solid.

## What I learned

`aiohttp.ClientResponse.headers` vs `.raw_headers` is a recurring trap when bridging aiohttp-typed responses into httpx-typed responses (or vice versa). The "decoded str view" of headers is convenient for logging but lossy across the ASCII boundary; the "raw bytes pairs" view round-trips faithfully. The general lesson is "for cross-stack header bridging, always use the rawest representation either side exposes" — which is also why this fix doesn't need to know what encoding aiohttp picked or what encoding httpx will pick: both sides handle bytes, so we just hand off bytes.
