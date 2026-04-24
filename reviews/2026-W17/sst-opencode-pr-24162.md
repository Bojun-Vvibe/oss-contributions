---
pr_number: 24162
repo: sst/opencode
head_sha: 6fc8560b380beb88b9ed5229f7e7963f04912b58
verdict: request-changes
date: 2026-04-24
---

# sst/opencode#24162 — desktop health check retry with exponential backoff

**What changed.** Replaces the old "loop forever, sleep 100ms" health-check in `packages/desktop/src-tauri/src/server.rs` with a bounded retry of `HEALTH_CHECK_MAX_ATTEMPTS = 6` calls to `check_health_with_retry`, each with a 2 s per-request timeout and a backoff sequence of 500/1000/2000/4000/4000 ms (`backoff_interval`, lines 56–60).

**Why it matters.** Old loop never exited on its own — the outer 30 s `select!` had to time out. New shape returns a typed `Result<(), String>` so the spawn task can log a real error.

**Concerns.**
1. **Loop has no return after exhaustion (lines 99–143).** The `for attempt in 0..HEALTH_CHECK_MAX_ATTEMPTS` body returns `Err` only when `attempt + 1 >= HEALTH_CHECK_MAX_ATTEMPTS`. If `HEALTH_CHECK_MAX_ATTEMPTS` is ever changed to 0, the function falls off the end of the `for` and returns nothing — Rust will reject the build today, but if you add an early `continue` later this becomes a footgun. Add a final `Err(...)` after the loop or use `(0..N).try_fold(...)` instead.
2. **`builder.no_proxy()` is now unconditional (line 93).** The old code was careful: only `no_proxy()` when host parses as loopback. Making it unconditional regresses the "user gave a non-loopback hostname" case — corporate proxies won't be honored even when needed. The PR comment says "always skip proxy for localhost health checks" but the function has no such gate; `hostname` is whatever the caller passed. Restore the loopback check.
3. **Budget math in the comment is wrong.** "500+1000+2000+4000+4000=11.5s" — that sums to 11.5 s only if you count one base sleep (it's actually 11.5 s, OK), but the per-request timeout is 2 s × 6 = 12 s, so worst case is ≈23.5 s — still under 30 s, but tight. Note that `tokio::time::timeout` of 2 s on `req.send()` is shorter than the `reqwest::Client::builder().timeout(7s)` (line 94), making the 7 s setting dead. Remove one or align them.
4. **`backoff_interval(attempt - 1)` (line 102)** is called with `attempt > 0`, so `attempt - 1` is safe — but uses unsigned subtraction. Document or `attempt.saturating_sub(1)` to be explicit.
5. **Lost detail in error.** `"Health check failed after all retries"` (line 142) drops the actual last error. Capture the last `e` and bubble it.

Retry shape is right; the unconditional `no_proxy()` and missing exhaustion-return need fixing before merge.
