# openai/codex #20751 — Bound websocket request sends with idle timeout

- **Repo:** openai/codex
- **PR:** #20751
- **Head SHA:** `9efa5ee246fd3b456da80c75f0c3d0f8b706e0b9`
- **Verdict:** merge-as-is

## Summary

Tiny, surgical fix. `responses_websocket.rs:559-566` previously called
`ws_stream.send(...).await` unbounded — if the socket write side stalled
(server-side disconnect not yet observed by the client), the task would
sit indefinitely because only the *receive* side had `idle_timeout`
applied. This wraps the send in `tokio::time::timeout(idle_timeout, ...)`
so a stalled write surfaces as `ApiError::Stream("idle timeout sending
websocket request")` and the caller's reconnect path engages.

## Specific notes

- **`codex-rs/codex-api/src/endpoint/responses_websocket.rs:559-566`:**
  the new error message ("idle timeout sending websocket request")
  cleanly distinguishes from the existing receive-side timeout.
  Telemetry/log greppers can tell which side stalled.
- The `.and_then(|result| result.map_err(...))` chain preserves the
  original `failed to send websocket request: {err}` formatting for
  the non-timeout case — no observability regression.
- `request_start = Instant::now()` (line 558) is captured before the
  timeout wrapper, so the existing `t.on_ws_request(...)` telemetry
  call still measures the full attempt duration including the timeout
  wait. Correct.

## Rationale

Two-line behavioural change with a precise bug story, no test churn
needed (timeouts on stalled writes are notoriously hard to fake in
unit tests and the existing receive-timeout pattern was unit-tested
the same way: trusted construction). The PR description names the
exact failure mode and reuses the existing `idle_timeout` config so no
new knob is introduced. Ship it.
