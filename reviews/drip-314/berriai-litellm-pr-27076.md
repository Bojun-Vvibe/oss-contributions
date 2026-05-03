# Review — BerriAI/litellm #27076: fix(exceptions): handle non-integer status_code in MidStreamFallbackError

- **PR:** https://github.com/BerriAI/litellm/pull/27076
- **Head SHA:** `ed1bc3f0d3ba14c8f7c50bd0c7e4cc961d0d7d80`
- **Author:** nileshpatil6
- **Files:** 2 changed (+33 / −2)
  - `litellm/exceptions.py` (+12/−2)
  - `tests/test_litellm/test_exception_header_preservation.py` (+21)

## Verdict: `merge-as-is`

## Rationale

Tight, well-scoped bugfix. Two `int(original_status)` call sites in `exceptions.py` (around lines 957 and 1004 in the diff) previously assumed that any non-`None` value at `getattr(original_exception, "status_code", None)` would be coercible to int. In practice some upstream provider exceptions surface a string sentinel like `"litellm_error"`, which raised `ValueError` inside the exception constructor — meaning the fallback chain never engaged and the proxy returned HTTP 500 instead of 503. The fix wraps both `int(...)` calls in `try/except (ValueError, TypeError)` and falls back to `503`, which matches the pre-existing default for the `None` branch. That's exactly the right shape: same default, same semantic, no behaviour change for the well-typed callers.

The test added at `tests/test_litellm/test_exception_header_preservation.py:257-276` exercises the precise regression scenario: a `_FakeException` with `status_code = "litellm_error"` (string), passed as `original_exception`, and asserts both `midstream_error.status_code == 503` and `midstream_error.response.status_code == 503`. That nails the second-order effect (the response object also carries the corrected status) which is the bit a naive fix could miss. Good test.

Catching `TypeError` in addition to `ValueError` is the right paranoia — a pathological `status_code` value of e.g. a list or a dict would otherwise crash the constructor for the same reason. No behaviour change for the existing happy path. Merge as-is.

## Banned-string check

Diff scanned; no banned tokens present.
