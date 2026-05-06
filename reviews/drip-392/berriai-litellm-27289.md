# BerriAI/litellm #27289 — Fix: Handle multipart/form-data with binary files in pass-through endpoint

- PR: https://github.com/BerriAI/litellm/pull/27289
- Head SHA: `6913383953c3622f26f6c93d2889a92af7cd8d18`
- Base: `litellm_internal_staging`
- Size: +121 / -21 across 2 files
- Files:
  - `litellm/proxy/pass_through_endpoints/pass_through_endpoints.py` (+17/-21)
  - `tests/test_litellm/proxy/pass_through_endpoints/test_pass_through_endpoints.py` (+104/-0)

## Verdict
**merge-after-nits**

## Rationale
Correct fix for a real 500 (`UnicodeDecodeError: 'utf-8' codec can't decode byte 0xc4`) when binary files are uploaded through pass-through with `Content-Type: multipart/form-data`. The previous code at `litellm/proxy/pass_through_endpoints/pass_through_endpoints.py:1192-` tried `await request.json()` first, which forces FastAPI/Starlette to read+decode the body as UTF-8; PDF/image bytes blow that up before the catch can do anything. Even if the catch fires, the body stream is already consumed, breaking downstream `make_multipart_http_request`.

The fix reduces the multipart branch to `pass` (lines 1195-1209 post-diff), with a clear comment explaining the rationale and the edge case (misconfigured client sending JSON with multipart content-type now has to fix its header — acceptable trade). The actual multipart parsing is delegated to `make_multipart_http_request`, which is the right layer.

Test coverage is proportionate:
- `test_parse_request_data_multipart_binary_file` (new, `test_pass_through_endpoints.py:2624-2670`) — pins the regression with PDF magic bytes + a `0xC4` continuation byte, the same byte from the bug report.
- `test_parse_request_data_json` (line 2674) and `test_parse_request_data_form_urlencoded` (line 2701) — guard the other two branches against future drift.

## Specific lines
- `pass_through_endpoints.py:1194-1209` — multipart branch is now a `pass` with comment. Correct.
- `pass_through_endpoints.py:60-63` — unrelated import re-ordering (alphabetized `EndpointType`). Trivial drive-by; fine.
- `pass_through_endpoints.py:2323-2326` — whitespace-only re-indent of an existing comment block in `_register_pass_through_endpoint`. **Nit:** this is unrelated and adds noise to `git blame`. Drop it from the PR if a re-roll is needed; otherwise harmless.
- `test_pass_through_endpoints.py:2639` — `bytes([0xC4, 0x80, 0x81, 0x82])` exactly mirrors the failure-mode byte. Good test design.

## Nits before merging
1. Drop the whitespace re-indent at lines 2323-2326 (no behavioural change, dirties blame).
2. Test docstring at `test_pass_through_endpoints.py:2627` references the original bug bytes — would be nice to also link the issue number once one is filed, but body says "Relevant issues" is empty. Worth filing one for traceability.
3. Consider also asserting in the test that `make_multipart_http_request` would still receive the unconsumed body (a `request.body()` / `request.stream()` mock check). Not strictly required since the contract is "skip parsing here".
