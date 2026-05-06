# openai/codex#21310 — [codex] add exec-server health check

- Head SHA: `1276cfc9c427d3bf73a0fd67b43f03773bcc973e`
- Author: @richardopenai
- Link: https://github.com/openai/codex/pull/21310

## Notes
- `codex-rs/exec-server/src/server/transport.rs:18–25` defines a hardcoded `HEALTH_RESPONSE` byte string with `connection: close`. Fine for a probe, but `content-length: 3` with body `"ok\n"` (3 bytes) is consistent — good.
- `serve_connection` at `transport.rs:148+` peeks `HTTP_REQUEST_PEEK_BYTES = 64` to disambiguate WS upgrade vs `GET /health`. Peek-then-route is the right pattern, but `stream.peek` on `TcpStream` requires the socket actually be a `TcpStream` (not a generic `AsyncRead`), so the listener path is fine — just confirm no future Unix-domain transport regresses here.
- `HEALTH_REQUEST_MAX_BYTES = 8 * 1024` cap on slurping the request line is a reasonable DoS guard.

## Verdict
`merge-after-nits`

Solid implementation. Add a test that a malformed `GET /health` (e.g. missing `\r\n\r\n` within 8KiB) terminates the connection cleanly rather than hanging, and document `connection: close` so callers don't pipeline.
