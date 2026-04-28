# PR #26706 — fix(github_copilot): invalidate stale access token on 401 and re-acquire

- **Repo:** BerriAI/litellm
- **Link:** https://github.com/BerriAI/litellm/pull/26706
- **Author:** iandvt (Ian Davies)
- **State:** OPEN
- **Head SHA:** `43f700ecd2329949a4bc05a7ed94a4cbaa865cdd`
- **Files:** `litellm/llms/github_copilot/authenticator.py` (+36/-2), `tests/test_litellm/llms/github_copilot/test_github_copilot_authenticator.py` (+401/-16)

## Context

Fixes issue #25312. The reported failure mode is: machine sleep/wake (or any server-side revocation) leaves the cached OAuth access token at `~/.config/litellm/github_copilot/access-token` invalid. `_refresh_api_key()` retries 3 times reading the *same* stale token from disk, then `RefreshAPIKeyError` propagates. Router retries the whole request, `get_access_token()` reads the same dead token again, and the proxy is dead until the user manually deletes the directory or restarts. There's no code path that maps "GitHub returned 401" → "delete the cached token and run device flow again."

## What changed

In `authenticator.py`:

- `_refresh_api_key()` now treats a 401 from `api.github.com/copilot_internal/v2/token` as a signal to call a new helper `_invalidate_and_reacquire_token()`. A `token_invalidated` boolean scoped to this call ensures we delete-and-reacquire **at most once** per refresh attempt.
- `_invalidate_and_reacquire_token()` (new): deletes the cached file, calls `get_access_token()` to trigger device flow, returns the fresh token or `None`. Critically, the flag is set `True` only when re-acquisition succeeds OR when `os.remove()` itself fails (because then re-acquire would just re-read the same stale file). When `os.remove()` succeeds but re-acquire fails (e.g., user dismisses device flow), the flag stays `False` so a subsequent 401 in the *same* loop iteration could try again — though in practice the loop will probably exit first.
- `get_api_key()` now also catches `GetAccessTokenError`, so a failed device flow from the new path doesn't escape as an unhandled exception.

The test file grows by 11 new tests on top of the existing 19 (30 total). The matrix is genuinely good: 401-then-success, persistent 401 (delete-once), 500-then-401 (delete still triggered), 500 alone (no delete), reacquire-fails (graceful degradation), `GetDeviceCodeError` during reacquire (same path), `os.remove` permission error (no infinite loop), 403 (does not delete — only 401), 401-then-500 (fresh token reused for the 500), `ConnectionError` (no delete), and `get_api_key` swallowing `GetAccessTokenError`.

## Design analysis

The "at most once per refresh attempt" gate is the load-bearing piece. Without it you'd have a delete-flap: every 401 deletes the file, device flow re-acquires a fresh token that the server *still* 401s on (because the actual problem isn't the token, it's a server-side block), and you cycle deletes forever. The flag pins the single delete to a single `_refresh_api_key()` call, and the loop's natural retry budget (3 attempts) bounds the worst case.

The decision not to use a `threading.Lock` is justified in the PR description (sync httpx, single-threaded per call; multi-worker is multi-process so a lock wouldn't help anyway). I agree, but: there's still a multi-process collision possible if two worker processes both 401 at the same instant and both try to delete the file. Whichever loses the race gets `FileNotFoundError` from `os.remove`, which the helper handles (returns `None`, flag stays `False`, next iteration re-attempts). That's actually the correct behaviour, but it's worth a comment near the `os.remove` saying so explicitly — otherwise a future contributor will look at the bare `try: os.remove`/`except OSError` and "fix" it.

The `get_api_key()` change to also catch `GetAccessTokenError` is the second-order fix and easy to miss in review. Without it, the new code path could surface a fresh exception type that callers weren't catching, regressing call sites that today only worry about `RefreshAPIKeyError`. Good catch.

## Risks

1. **Device flow inside a server request.** Re-acquisition triggers `get_access_token()`, which in user-facing CLIs prompts an interactive device code. Inside a litellm-proxy serving HTTP requests, that prompt would deadlock or write to a log nobody's reading. Worth confirming the device-flow path has a non-interactive mode when called from the proxy context, or that the proxy has a different code path entirely. The test uses mocks, so it doesn't expose this.

2. **Race with concurrent in-flight requests.** Two concurrent requests both 401, both invalidate, both run device flow simultaneously. One overwrites the other's fresh token. The PR doesn't claim to fix concurrent invalidation; the worst case is one extra device-flow round-trip, which is benign.

3. **Telemetry blind spot.** A token that gets re-acquired silently masks the underlying problem (server-side revocation? user changed orgs?). One log line at INFO when invalidation triggers would be cheap and high-value for ops.

## Verdict

**Verdict:** merge-after-nits

Add an INFO log at the invalidation site; add a comment near the `os.remove` failure path documenting the multi-process race semantics; confirm device-flow behaviour when called from the proxy server context (not interactive). The fix itself is well-shaped and the test matrix is excellent.

---

*Reviewed by drip-155.*
