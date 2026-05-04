# BerriAI/litellm #27075 — fix(exceptions): handle non-integer status_code in MidStreamFallbackError

- SHA: `c7ebbf116f4838a5f97fe68d6066c1e1688b2832`
- State: CLOSED, +8/-2 in `litellm/exceptions.py`

## Summary

Wraps two `int(original_status)` casts inside `MidStreamFallbackError.__init__` (and a sibling re-init path) in `try/except (ValueError, TypeError)`, defaulting to `503` on cast failure. Prevents an exception-during-exception when an upstream provider returns a non-numeric `status_code` attribute (e.g., a string like `"unknown"` or `None` shaped oddly).

## Notes

- `litellm/exceptions.py:957-961` and `:999-1003`: identical pattern in two places. Logic is correct: `getattr(original_exception, "status_code", None)` already returns `None` → 503; the new `try/except` catches the case where the attribute exists but is not int-coercible.
- Catching only `ValueError, TypeError` is the right scope — won't swallow `KeyboardInterrupt` etc.
- The two duplicated blocks suggest extracting a small helper `_coerce_status(original_status, default=503) -> int` would deduplicate, but for a defensive fix this size it's acceptable.
- PR is CLOSED — likely superseded or rejected. If superseded, the replacement should carry the same fix pattern.

## Verdict

`merge-after-nits` (had it stayed open) — extract `_coerce_status` to avoid the duplicated try/except. The behavior is correct as written. Since the PR is closed, this review is informational; track whether the successor PR (if any) preserves the defensive pattern.