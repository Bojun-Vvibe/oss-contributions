# BerriAI/litellm PR #27076 — fix(exceptions): handle non-integer status_code in MidStreamFallbackError

- Author: nileshpatil6
- Head SHA: `ed1bc3f0d3ba14c8f7c50bd0c7e4cc961d0d7d80`
- Diff: +33 / -2 across `litellm/exceptions.py` and `tests/test_litellm/test_exception_header_preservation.py`
- Fixes #26913

## Observations

1. **`litellm/exceptions.py:957-963`** wraps the initial `int(original_status)` call inside `MidStreamFallbackError.__init__` with `try / except (ValueError, TypeError)` defaulting to `503`. Catching both exception types is correct — `int(None)` raises `TypeError` (already short-circuited by the `is not None` guard, but defensive), `int("litellm_error")` raises `ValueError`. Default of `503 Service Unavailable` lines up with how `MidStreamFallbackError` is conceptually a "we lost the upstream stream, try a fallback" signal.
2. **`litellm/exceptions.py:1004-1010`** repeats the identical guard at the second `int()` callsite (the post-`super().__init__()` restoration). Bug report and fix description correctly identify both lines (958, 1000 in the original file). Slight smell — two identical try/except blocks five lines apart is ripe for a small `_coerce_status(value: Any, default: int = 503) -> int` helper. Not a blocker, but a cleaner shape if the author wants to take it.
3. **Test at `test_exception_header_preservation.py:257-277`** constructs a `_FakeException` with `status_code = "litellm_error"` (the exact failure-mode string from the linked issue), passes it as `original_exception`, and asserts both `midstream_error.status_code == 503` and `midstream_error.response.status_code == 503`. The dual assertion is good — it catches the case where only one of the two `int()` callsites is patched.
4. **Coverage gap**: no negative test for the `TypeError` branch (e.g. `status_code = object()` or a non-coercible custom type). The `ValueError` path is the realistic one but a parametrized test covering both branches would future-proof against someone "simplifying" to just `except ValueError:`.
5. **No release-notes / changelog entry** referenced. For an exceptions module change that affects fallback chain triggering, it's worth a CHANGELOG line so users debugging "why didn't my fallback fire?" can grep history.

## Verdict: `merge-after-nits`

Correct, minimal fix with a sharp test that reproduces the reported `'litellm_error'` string exactly. Pre-merge: extract the duplicated try/except into a 3-line `_coerce_status` helper, parametrize the test to cover both `ValueError` and `TypeError`, and add a CHANGELOG line so the fix is discoverable.
