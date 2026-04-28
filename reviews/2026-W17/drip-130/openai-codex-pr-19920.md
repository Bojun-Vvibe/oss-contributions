# openai/codex #19920 — Allow large remote app-server resume responses

- URL: https://github.com/openai/codex/pull/19920
- Head SHA: `dfe51b1078492abdcb1a6b106964abcf9d287332`
- Files: 2 (`codex-rs/app-server-client/src/lib.rs`, `src/remote.rs`)
- Size: +121 / −38
- Fixes #19837

## Summary

Two intertwined fixes for the remote TUI resume path: (1) raise the
tungstenite WebSocket frame/message cap from the 16 MiB default to 128 MiB
so a single-frame `thread/resume` JSON-RPC response carrying a long saved
session deserializes instead of being rejected at the transport layer; (2)
preserve the *concrete* worker-exit error (`BrokenPipe` / `InvalidData`)
when completing pending requests after a transport failure, instead of the
generic channel-closed error that callers were seeing previously.

## Specific references

- `remote.rs:59` introduces `REMOTE_APP_SERVER_MAX_WEBSOCKET_MESSAGE_SIZE: usize = 128 << 20`
  with a one-line justification comment naming "remote resume responses can
  legitimately carry large thread histories". The constant is only the cap,
  not the steady-state allocation — tungstenite still streams chunks, so
  the per-connection memory cost is unchanged for typical sessions.
- `remote.rs:175-189` switches `connect_async` → `connect_async_with_config`
  with `WebSocketConfig::default().max_frame_size(Some(...)).max_message_size(Some(...))`
  and explicit `disable_nagle: false` (preserves prior TCP behavior — the
  comment marker is the right kind of self-documenting boilerplate for a
  config struct that's easy to misset).
- `remote.rs:215` introduces `let mut worker_exit_error: Option<(ErrorKind, String)> = None`
  and three subsequent assignment sites at `:255` (write-failed →
  `BrokenPipe`), `:382` (rejection-write-failed → `BrokenPipe`), and `:395`
  (invalid JSON-RPC → `InvalidData`). The pattern is "set on the path that
  causes the worker to break out of its select loop, then drain pending
  requests with that concrete error" — which is the right inversion of the
  prior "drain with a generic error and lose the cause" shape.
- `lib.rs:1399-1450` is the regression test
  `remote_typed_request_accepts_large_single_frame_response`. Padding is
  `(17 << 20) + 1024` = 17 MiB + 1 KiB, deliberately above the prior 16 MiB
  default cap so the test will fail loudly if anyone reverts the config.

## Risk

The PR description honestly calls out "this isn't a perfect fix. It really
just moves the limit to a much larger value." That's correct and the right
trade-off given the alternatives the author surveyed. The concrete risks:

1. **128 MiB is large enough to be a denial-of-service knob if a hostile or
   buggy server emits an oversized frame** (e.g., a misconfigured server
   that streams binary data into the JSON-RPC channel). The cap protects
   against unbounded growth but a coordinated 128 MiB allocation per
   connection × N reconnect attempts is still painful. Worth a follow-up
   that streams the response into a bounded-size deserializer rather than
   buffering whole-frame.
2. **The fix masks the underlying server-side bug**: `thread/resume`
   should probably page large histories rather than ship them in one
   response. Adding a server-side issue reference in the PR or a TODO
   pointing at "switch resume to paginated stream" would prevent this from
   becoming a permanent design.

## Nits (non-blocking)

- The error-preservation pattern uses `Option<(ErrorKind, String)>` which
  is fine but a tiny named struct (`WorkerExitError { kind, message }`)
  reads better at the assignment sites and at the eventual drain site
  (which isn't visible in the truncated diff but presumably exists past
  `:395`).
- The duplicated `format!("remote app server at \`{websocket_url}\` write failed: {err_message}")`
  appears at `:243-245` and `:374-376` — extracting a `format_write_failure_msg(websocket_url, &err_message)`
  helper would prevent the two sites from drifting on a future "let's add
  the request_id to the message" change.
- The test sends one big response but doesn't exercise the new
  `worker_exit_error` preservation path. A second test that drops the
  websocket mid-response and asserts the pending-request future resolves
  with `BrokenPipe` (not generic channel-closed) would pin behavior (2).

## Verdict

`merge-after-nits` — the cap raise is correct, the error-preservation
refactor is a real quality improvement, and the regression test
deliberately straddles the old limit. The unaddressed concerns are
documented as follow-ups, not blockers.
