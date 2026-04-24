# PR-24138 — fix(desktop): retry + exponential backoff for sidecar health check

[sst/opencode#24138](https://github.com/sst/opencode/pull/24138)

## Context

`packages/desktop/src-tauri/src/server.rs` previously polled the local
sidecar's `/global/health` endpoint in an unbounded `loop { sleep(100ms);
check_health(...) }`. On slow boots (cold disk, large workspace,
antivirus scan in flight), the IDE host's outer 30 s timeout would fire
before the sidecar finished initializing. This PR replaces the spin-poll
with a fixed-budget retry: 6 attempts, exponential backoff capped at
4 s (`HEALTH_CHECK_BASE_INTERVAL_MS=500`, `HEALTH_CHECK_MAX_INTERVAL_MS=
4000`), each request gated by a 2 s `tokio::time::timeout`, total
budget ~23.5 s — fits inside the caller's 30 s window. The `reqwest::
Client` is now built once outside the loop, and `no_proxy()` is
applied unconditionally (was previously gated on a hostname check for
`localhost`/loopback IPs).

## Strengths

- Retry math is annotated in a comment that survives review: the
  ~23.5 s ceiling is computed and justified against the caller's 30 s
  budget. That's the kind of arithmetic that gets dropped a year later
  unless it's spelled out.
- `backoff_interval` is `#[inline]` and uses `saturating_mul` /
  `saturating_pow` — no panic on overflow even if `HEALTH_CHECK_MAX_
  ATTEMPTS` is bumped to a silly value.
- Per-attempt `tokio::time::timeout` is layered on top of `reqwest`'s
  own 7 s client-level timeout. That's belt-and-suspenders, but the
  outer 2 s wrap is what actually keeps a stuck TCP connect from
  burning the entire budget on a single attempt.
- Connection pool is built once before the loop, so retries reuse the
  same client and benefit from keep-alive.

## Concerns / risks

- **`no_proxy()` is now unconditional.** Previously the code only
  bypassed proxy env vars for `localhost`/loopback hostnames; now it
  bypasses for every URL passed in. The function name is
  `check_health_with_retry(url, ...)` — there's nothing constraining
  the caller to pass a loopback URL. If a future caller points this at
  a non-local host (LAN sidecar, remote bridge), corporate proxy
  policy will be silently bypassed. Worth either renaming the function
  to encode the loopback assumption (`check_local_health_with_retry`)
  or restoring the conditional.
- **Silent timeout interaction.** The client has a 7 s
  `reqwest::Client::builder().timeout(...)`, but each request is also
  wrapped in `tokio::time::timeout(Duration::from_millis(2000), ...)`.
  The 2 s outer always wins, making the 7 s setting dead code. Either
  drop the inner client timeout or align them — leaving both invites
  the next reader to assume one of them is the active limit.
- **Final error is a stringly-typed `"Health check failed after all
  retries"`.** The last per-attempt failure (timeout vs network error
  vs non-2xx) is logged at `debug!` but not threaded into the returned
  `Err`. From the caller's perspective every failure mode looks
  identical, which is exactly what the original code did — this PR
  could have surfaced a structured cause.
- **Backoff sequence has a quirk.** `backoff_interval(attempt)` with
  `attempt.min(3)` produces 500, 1000, 2000, 4000, 4000 ms. The fifth
  retry's wait is identical to the fourth. Probably intentional
  (clamp), but `(2000 → 4000 → 4000)` reads like a typo at a glance —
  worth a comment that the cap is by design.
- **No jitter.** If the desktop relaunches the sidecar after a crash
  loop and many such app instances exist on the same host (multi-user
  shared box, dev container fleet), all retries will land on the same
  500/1000/2000/4000 ms grid. Adding `±20%` jitter would be cheap.
- **No test changes** in the diff. The retry math, backoff clamp, and
  early-exit-on-success branches deserve a unit test that mocks the
  HTTP layer; otherwise the next refactor of `backoff_interval` ships
  unguarded.

## Verdict

**Approve with caveats.** Net improvement over the spin-poll, and the
budget arithmetic is sound. Before merge: either constrain
`no_proxy()` to loopback or rename the function; reconcile the dual
timeout values; and add at least one regression test for the backoff
sequence so the `attempt.min(3)` clamp doesn't get refactored away
silently.
