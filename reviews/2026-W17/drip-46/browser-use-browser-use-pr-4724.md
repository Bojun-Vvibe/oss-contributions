# browser-use/browser-use #4724 — fix: implement retry logic for captureScreenshot to avoid timeouts

- **PR:** https://github.com/browser-use/browser-use/pull/4724
- **Head SHA:** `33e0662369050615fe621845b9990724d27fdd1e`
- **Files changed:** 1 — `browser_use/browser/session.py` (+22/−7).

## Summary

Wraps the `Page.captureScreenshot` CDP call in a 3-attempt retry loop with linear backoff (1s, 2s) to mitigate transient timeouts and flake reported in #4651. On the first two failures, logs a warning and sleeps; on the third, re-raises an `Exception` with the last error string interpolated. Empty-data responses now also feed the retry loop (previously they were a single hard-fail).

## Line-level call-outs

- `browser_use/browser/session.py:3925-3930` — bare `except Exception as e:` catches *everything*, including `asyncio.CancelledError` on Python ≤3.7 semantics, and on 3.8+ still catches `KeyboardInterrupt` if it inherits via subclassing. More importantly, it catches non-retryable error classes (e.g. invalid `CaptureScreenshotParameters` from the user's own input, or session-closed errors that will never recover) and treats them as transient, masking the failure for ~3 seconds before re-raising. Recommend either: (a) classify retryable vs non-retryable by exception type (`TimeoutError`, `asyncio.TimeoutError`, CDP-timeout-shaped exceptions only) and let the rest fall through immediately, or (b) at minimum, exclude `asyncio.CancelledError` so a cancelled task doesn't get retried.
- `:3934` — `await asyncio.sleep(1.0 * (attempt + 1))` on `attempt = 0,1` gives 1s and 2s sleeps — wall-clock 3s of retry budget. For an action loop that may take screenshots dozens of times per task, a 3s tail on a permanent failure (e.g. session detached) adds visible latency. Worth either parameterizing the backoff or dropping it to 0.5s/1s.
- `:3917-3938` — the `result` capture, `if not result or 'data' not in result: raise` check, and the `base64.b64decode(result['data'])` call are now all inside the retry block. That means a `b64decode` failure on a corrupted-but-present `data` field also triggers a retry — which is probably wrong (the same corruption will reappear on the next call) but harmless. The `Path(path).write_bytes(...)` call is *also* inside the retry block — if the first attempt succeeds at decoding but `write_bytes` raises (e.g. disk full, permission denied), the retry loop will re-call CDP, get a new screenshot, and try to write it again. The disk error won't go away. The write should arguably happen *outside* the retry: retry the capture, succeed, then write once.
- `:3941` — final raise: `Exception(f'Screenshot failed after {max_retries} attempts. Last error: {last_exception}')`. Loses the original exception's traceback chain. `raise Exception(...) from last_exception` (or `raise last_exception` to preserve the original entirely) gives the caller a real stack trace instead of an opaque string. The current form makes downstream error handling and observability strictly worse.
- General: no test added. The existing test suite presumably has at least one screenshot-path test that will exercise the happy path; a new test that simulates two transient `TimeoutError`s followed by a success would pin the retry-then-succeed contract that this PR introduces. Without it, a future refactor can silently flip `max_retries = 1` and nothing fails.
- General: `max_retries = 3` and the `1.0 * (attempt + 1)` cadence are hard-coded inside the function. `take_screenshot` is called in tight loops in many agents; making these class-level constants (or `BrowserSession` attributes with sensible defaults) would let users tune the trade-off without forking. Not blocking, but worth noting.
- The bare-`Exception('No data returned…')` raised at `:3922` to feed the retry loop (when the CDP responds with `{}` or missing `data`) is reasonable — that *is* a transient-shaped failure (CDP sometimes returns empty responses during navigation). Good.

## Verdict

**request-changes**

## Rationale

The intent — make screenshot capture resilient to transient CDP timeouts — is correct, and a bounded retry loop is the right shape. But the implementation has three concrete problems that should land before merge: (1) the bare `except Exception` makes non-transient failures visibly slower without changing their outcome, and risks swallowing `CancelledError`; (2) `Path(path).write_bytes(...)` lives inside the retry, so a one-time disk error becomes three CDP calls plus three failed writes; (3) the final raise discards the original traceback, making the bug class strictly harder to debug than it was before. All three are small fixes. Add one regression test that proves "two transient timeouts → succeeds on attempt 3", split the file write out of the retry block, narrow the except, and chain the final exception with `from`. After that this is a clean reliability improvement.
