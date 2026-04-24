# BerriAI/litellm PR #26388 — fix: bedrock guardrail sse streaming exception

- **Repo:** BerriAI/litellm
- **PR:** [#26388](https://github.com/BerriAI/litellm/pull/26388)
- **Head SHA:** `b99c44af93c8ebd0c7f3d7029a6eccc7f3ea9383`
- **Size:** +200/-46 across 2 files (one prod, one test)
- **Reviewer:** Bojun (drip-23)

## Summary

Wraps the entire body of
`BedrockGuardrail.async_post_call_streaming_iterator_hook` in a
`try / except HTTPException` so a `_get_http_exception_for_blocked_
guardrail`-shaped raise during streaming does not escape the async
generator after the HTTP 200 + SSE headers have already been written.
On catch, two SSE frames are emitted in band: a `data: {error: ...}`
frame and the conventional `data: [DONE]\n\n` terminator.

This is the textbook async-generator-cannot-raise-after-headers fix.
The pre-PR code propagated the exception, which FastAPI/Starlette
serialized into a broken/truncated stream the OpenAI SDK and most
clients couldn't parse — they'd surface a generic
"unexpected end of stream" or JSONDecodeError instead of the actual
guardrail violation message.

## What's changed

### `litellm/proxy/guardrails/guardrail_hooks/bedrock_guardrails.py`

Diff lines 5–187. Three substantive edits:

1. **Hoist a `try:` over the `if isinstance(assembled_model_response,
   ModelResponse):` branch.** The `if/else` body that was previously
   un-protected (diff lines 9–8 of the pre-image) is now the body of
   `try:` (new line 34). The `else: for chunk in all_chunks: yield
   chunk` non-validation branch is also inside the try (diff lines
   166–168), which is fine — that branch can't raise the
   `HTTPException` because it doesn't call `make_bedrock_api_request`.

2. **Add the `except HTTPException as e:` handler** (diff lines
   168–186). The handler:
   - extracts `e.detail.get("error", str(e.detail))` if `e.detail` is
     a dict (matching the production shape from
     `_get_http_exception_for_blocked_guardrail`, which raises with
     `detail={"error": "...", "bedrock_guardrail_response": "...",
     "guardrailIdentifier": "...", "guardrailVersion": "..."}`);
   - falls back to `str(e.detail)` for non-dict shapes;
   - constructs an OpenAI-style error envelope:
     `{"error": {"message": ..., "code": e.status_code, "type":
     "guardrail_violation"}}`;
   - yields one `data: <json>\n\n` frame and one
     `data: [DONE]\n\n` terminator.

3. **Re-indent the entire validation body** (diff lines 35–166) to
   sit one level deeper inside the new try. This is the bulk of the
   line count and is whitespace-only — but the diff is large enough
   to make code review noisy. Worth requesting that the maintainer
   re-review with `git diff -w` to filter the indentation noise and
   focus on the real change.

### `tests/test_litellm/proxy/guardrails/guardrail_hooks/test_bedrock_guardrails.py`

Diff lines 190–346. Two new tests, one per code path:

- `test_streaming_hook_converts_http_exception_to_sse_error_frame_
  parallel_path` (lines 222–284): post_call event hook (no pre/during)
  → `should_validate_input == True` → asyncio.gather path. Patches
  `make_bedrock_api_request` to `AsyncMock(side_effect=hard_block)`,
  iterates the generator, asserts exactly 2 SSE frames out, asserts
  the error payload's `code == 400`, `type ==
  "guardrail_violation"`, and `message == "Violated guardrail
  policy"` (a clean string, not a Python dict repr — that's the
  important assertion at line 284).

- `test_streaming_hook_converts_http_exception_to_sse_error_frame_
  output_only_path` (lines 287–346): during_call event hook →
  `should_validate_input == False` → single-call OUTPUT-only branch.
  Same assertions.

Both tests use the **production dict-detail shape** for the
HTTPException (lines 252–261 and 315–324), which is the right
choice — they pin against what `_get_http_exception_for_blocked_
guardrail` actually raises in the wild, not against a synthetic
exception shape that would let bugs slip through.

## Concerns

1. **`HTTPException` is the only catch — bare `Exception` from
   bedrock SDK errors still escapes**

   The handler catches `HTTPException` only. If
   `make_bedrock_api_request` raises a non-HTTPException — e.g., a
   `botocore.exceptions.ClientError` from a transient AWS API
   failure, or a `GuardrailInterventionNormalStringError` that the
   `try/except GuardrailInterventionNormalStringError` blocks above
   already handle but a different exception class slipping through —
   the bug recurs (exception escapes the generator post-headers).
   The fix should arguably be `except (HTTPException, Exception)`
   with separate handling, or at minimum a broad `except Exception`
   fallback that emits a generic SSE error frame for the unknown
   case. Otherwise this PR fixes the *known* leak but leaves the
   *class* of bug intact.

2. **No `import json` shown in the prod diff**

   Diff line 185 calls `json.dumps(error_data)` but the import isn't
   visible in the hunk. The file likely already imports `json`
   elsewhere (the test file imports it locally as `_json` at line
   231) — but worth confirming. If the prod file imports json only
   inside a function or under a `TYPE_CHECKING` guard, this PR
   needs an explicit top-level `import json`.

3. **`# type: ignore` on the yields**

   Diff lines 185–186 have `# type: ignore` on both yields. The
   generator is typed (probably) as `AsyncIterator[ModelResponseStream]`
   but the new yield is a `str` (the SSE frame). The right fix is
   probably to widen the return type to `AsyncIterator[Union[
   ModelResponseStream, str]]` or to wrap the SSE frame in a
   `ModelResponseStream` with the error in a delta — not just
   silence the type checker. The current shape is the classic
   "wire-format bypass" — the consumer side has to know to look for
   string yields specially, which is fragile.

4. **`should_validate_input` rebound only over try**

   The `should_validate_input` local at diff line 44 and `output_
   guardrail_response` at line 65 are now declared inside the try.
   If a future refactor moves the asyncio.gather try-except (current
   lines 96–102) to surface the error elsewhere, those locals lose
   their narrow scope. Mostly stylistic — Python doesn't enforce
   block scope — but worth a brief comment explaining the structure.

5. **`MockResponseIterator` initialized inside the validation
   branch — not the early-exit case**

   Diff line 152: `mock_response = MockResponseIterator(
   model_response=assembled_model_response)`. If
   `assembled_model_response` is *not* a `ModelResponse` (the `else`
   branch starting at line 165), the original chunks are forwarded
   unmodified, but they're still inside the new try. If yielding one
   of those original chunks somehow raised an HTTPException
   (extremely unlikely but possible if a wrapping middleware
   inspected them), the catch would now activate and overwrite a
   normal stream with an error envelope. Not a real concern in
   practice; just noting the broader catch-scope.

6. **Two near-identical tests differ in only the `event_hook`
   parameter — could be `pytest.mark.parametrize`**

   Lines 222–284 and 287–346 are 90% the same code. Parametrizing
   on `event_hook=[GuardrailEventHooks.post_call,
   GuardrailEventHooks.during_call]` would halve the test code and
   make the matrix more obvious. Optional.

## Verdict

`merge-after-nits` —

- broaden the catch to either `(HTTPException, Exception)` with
  a generic SSE error frame for the unknown case, or document
  explicitly why HTTPException is the *only* leak class worth
  catching;
- confirm/add a top-level `import json` in the prod file;
- consider widening the generator's type or wrapping the SSE error
  frame in a proper response object instead of two `# type: ignore`
  raw-string yields;
- parametrize the two tests.

The fix is real, the tests pin the right behavior with production-
shaped exceptions, and the SSE error envelope shape (`error.code`,
`error.type`, `error.message`) matches the OpenAI SDK error contract
so existing client error handling will work.

## What I learned

The "async generator cannot raise after the response headers are
written" pitfall is recurring in litellm's streaming guardrail and
auth layers — same shape, different file. A single shared decorator
`@safe_streaming_generator` that catches a configurable exception
class and emits an SSE error frame would prevent the next instance
of this bug. Worth filing as a follow-up: the fix here is local,
but the bug pattern is global.
