# BerriAI/litellm PR #26448 — fix(content_filter): log guardrail_information on streaming post-call

- **Repo:** BerriAI/litellm
- **PR:** [#26448](https://github.com/BerriAI/litellm/pull/26448)
- **Head SHA:** `e6fdf5a17317443d05c2ea9b9207430446d659b1`
- **Author:** michelligabriele
- **Size:** +381/-63 across 2 files
- **Reviewer:** Bojun (drip-24)

## Summary

Closes a streaming-vs-non-streaming parity gap in
`ContentFilterGuardrail.async_post_call_streaming_iterator_hook`.
On non-streaming requests, `apply_guardrail`'s `finally` block
writes a `standard_logging_guardrail_information` entry to
`request_data["metadata"]`, which propagates to `SpendLogs`,
the SLO, and the UI Request Lifecycle panel. On streaming
requests, the iterator hook never wrote this entry, so the UI
showed no post-call step and observability consumers saw nothing.

The fix wraps the iterator body in `try/except/finally`, threads
a per-call `detections` list through `_filter_single_text(...)`,
and writes the log entry from `finally` on all three exit paths
(clean / MASK / BLOCK).

The PR is well-described, well-scoped, and ships three new
tests covering the three exit paths.

## Key changes

### `content_filter.py` (+~80/-65)

The wrapping shape at line ~1880:

```python
start_time = datetime.now()
detections: List[ContentFilterDetection] = []
masked_entity_count: Dict[str, int] = {}
status: GuardrailStatus = "success"
exception_str: str = ""

try:
    async for item in response:
        ...
        try:
            detections.clear()
            masked_text = self._filter_single_text(text_to_check, detections=detections)
            ...
        except HTTPException:
            raise
        ...
except HTTPException:
    status = "guardrail_intervened"
    raise
except Exception as e:
    status = "guardrail_failed_to_respond"
    exception_str = str(e)
    raise e
finally:
    self._count_masked_entities(detections, masked_entity_count)
    self._log_guardrail_information(
        request_data=request_data,
        detections=detections,
        status=status,
        start_time=start_time,
        masked_entity_count=masked_entity_count,
        exception_str=exception_str,
    )
```

Three things to note:

1. **`detections.clear()` before each scan.** This is the right
   call. The author's comment explains it: `_filter_single_text`
   scans the entire accumulated buffer every chunk, so previous-
   chunk matches would be re-found and double-counted in
   the final log row. Clearing on each iteration keeps only the
   final scan's detections, which is what gets logged.

2. **BLOCK still records correctly because handlers append to
   detections before raising.** This is a load-bearing
   invariant — if `_filter_single_text` ever raised
   `HTTPException` *before* appending to detections, the BLOCK
   log row would be empty. Worth a comment in
   `_filter_single_text` ("must append to detections before
   raising") to make this explicit.

3. **`raise` vs `raise e` inconsistency.** The `HTTPException`
   re-raise uses bare `raise`, the bare-Exception arm uses
   `raise e`. Both are correct, but `raise` (without `e`)
   preserves the traceback better. The bare-Exception arm
   should be `raise` too.

### Test coverage (`test_content_filter.py`, +285)

Three new tests in `TestContentFilterGuardrail`:

1. `test_streaming_hook_logs_guardrail_information_clean` — no
   detections fire, asserts `entries[0]["guardrail_status"] ==
   "success"` and `entries[0]["guardrail_response"] == []`.
2. `test_streaming_hook_logs_guardrail_information_mask` — MASK
   pattern fires, asserts the success row contains the detection
   and `masked_entity_count` is non-zero.
3. `test_streaming_hook_logs_guardrail_information_block` —
   BLOCK pattern raises `HTTPException`, the `finally` block
   still writes a row with
   `guardrail_status == "guardrail_intervened"`.

The PR author claims all 51 tests in `test_content_filter.py`
plus the broader 1209-test guardrails suite pass locally.

## Concerns

1. **`raise` vs `raise e` in the bare-Exception arm.**

   ```python
   except Exception as e:
       status = "guardrail_failed_to_respond"
       exception_str = str(e)
       raise e  # ← should be `raise`
   ```

   `raise e` resets the traceback in some Python versions and
   loses the implicit chaining. Cosmetic, but worth fixing.

2. **`detections.clear()` is a per-iteration mutation that
   makes the contract subtle.**

   The comment is good, but a future contributor might rip the
   clear out thinking they're fixing a "bug". A defensive
   alternative: use a local `iteration_detections = []` per
   chunk, then on `is_final` set the outer `detections` to the
   final iteration's value. Slightly more code but harder to
   mis-refactor. Not blocking.

3. **`_count_masked_entities` is called in `finally` even on
   the early-exit-with-no-content path.**

   If the upstream stream is empty (no chunks), `detections`
   stays `[]`, `_count_masked_entities` is a no-op, and the log
   row records `success` with no detections. That's the
   intended behavior — but worth verifying that
   `_count_masked_entities` doesn't have a precondition like
   "detections must be non-empty".

4. **The PR explicitly notes `AimGuardrail` and
   `_OPTIONAL_PresidioPIIMasking` have the same gap.**

   The author flagged that the same structural gap exists in
   `aim.py` and `presidio.py` and intentionally left them out
   of scope. This is the right call for PR sizing, but a shared
   `_log_streaming_guardrail_information` decorator/helper
   would be the cleaner long-term fix. Should file a follow-up
   issue.

5. **`request_data["metadata"]` mutation is the side-channel
   contract.**

   `_log_guardrail_information` writes into
   `request_data["metadata"]["standard_logging_guardrail_information"]`,
   and downstream the SpendLogs row + SLO read from there. This
   side-channel is fragile — if anyone in between
   reassigns `request_data["metadata"] = {...}`, the log row is
   silently dropped. Worth a doc-comment on
   `_log_guardrail_information` calling out the mutation
   semantics.

## Verdict

`merge-as-is` — high-quality PR. The bug is real (streaming
post-call guardrail observability was silently empty), the fix
is minimal and correctly scoped, the test coverage hits all
three exit paths, and the author proactively flagged the
out-of-scope sibling files (`aim.py`, `presidio.py`). The
`raise e` → `raise` and the follow-up issue for the shared
helper are nice-to-haves but not blockers.

## What I learned

The "streaming has different observability than non-streaming"
class of bug is endemic to systems that separate per-request
hooks from per-chunk hooks. The non-streaming path runs
`apply_guardrail` with a `finally` block that writes the log
row; the streaming path runs the iterator hook which has its
own loop and forgot to mirror the `finally`. The clean
architectural fix is a single decorator that both paths use:
`@with_guardrail_logging` that handles the start_time / status /
detections accumulation invariants in one place. The lesson:
when you have two code paths that should produce the same
side-effect, factor the side-effect into a single shared
helper, not into "remember to do X in both places". This PR
is the right tactical fix, but the strategic fix is one level
of abstraction up.
