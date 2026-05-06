# anomalyco/opencode#25959 — fix(server): emit keep-alive newlines during /session/:id/message

- **Head SHA**: `4c69da8236f1d4786d33373deae69b45ae53ec8e`
- **Stats**: +20 / -2, 1 file (`packages/opencode/src/server/instance/session.ts`)

## Summary

The `POST /session/:sessionID/message` route awaits `SessionPrompt.prompt(...)` to fully complete before writing any byte to the response stream. For long synchronous tool calls (multi-step `kubectl`/`git`, model latency over slow links, 60+ minute compactions) the response stream stays silent the entire time, and HTTP clients with finite per-recv timeouts (httpx defaults to 5s; some workflow runners cap at minutes) hit `ReadTimeout` before the first byte arrives. This PR adds a 30s `setInterval` that writes a single `\n` to keep the connection producing traffic — exploiting the fact that JSON parsers ignore leading whitespace before a value, so the response stays valid `application/json`.

## Specific citations

- `packages/opencode/src/server/instance/session.ts:858-864` (added): `setInterval` fires every `30_000` ms and calls `stream.write("\n").catch(() => {})`. The `.catch(() => {})` is load-bearing — if the client has already disconnected, `stream.write` will reject and we don't want the keepalive interval to crash the route handler. 30s is a defensible choice against httpx's 5s default (well below) and most ALB/nginx idle timeouts (60s+).
- `:865-870` (try/finally): the `clearInterval` runs in *both* the happy path (line 867 immediately after `await SessionPrompt.prompt`) AND the `finally` block (line 869) — this is double-clear-safe (`clearInterval` on an already-cleared id is a no-op) and the explicit pre-await clear is the right pattern because it stops the keepalive from racing with the final JSON write.
- `:867`: switched from `stream.write(JSON.stringify(msg))` (no await) to `await stream.write(JSON.stringify(msg))` — strictly correct since the previous fire-and-forget could let the function return before the final body bytes flushed; the await closes that gap. Drive-by improvement worth calling out in the PR body.
- The whitespace-prefix-tolerance claim is true for `JSON.parse` and Python `json.loads`, but **not** for streaming JSON parsers that match `{` as the first non-whitespace token only when followed by a key — anything tracking byte offsets against the raw stream (e.g. some HTTP middleware that records the byte where the JSON starts) will see the offset shift by N bytes per emitted newline. Low risk but worth noting.

## Verdict

**merge-after-nits**

## Rationale

Solves a real protocol gap: the silent-stream-then-ReadTimeout class of bug is exactly what kills long tool calls behind any non-trivial HTTP plumbing. The 30s cadence + `\n` payload + JSON-leading-whitespace-tolerance trick is a well-known idiom and correctly implemented; the try/finally double-clear is safe and the await on the final write is a quiet correctness improvement. Three nits: (1) the `30_000` literal should be a named constant (`KEEPALIVE_INTERVAL_MS`) at module scope so it can be tested and overridden via `OPENCODE_EXPERIMENTAL_*` like the bash timeout pattern in `prompt.ts`; (2) no test coverage — a fixture test that mounts the route, holds `SessionPrompt.prompt` open for >30s via a controllable promise, and asserts the response body contains a leading `\n` would lock the wire contract; (3) `Content-Type` header is not explicitly set anywhere visible in the diff, so confirm Hono's `stream(...)` default isn't `text/event-stream` (which would make the `\n`-as-whitespace claim void since SSE has different framing rules). None block merge; the implementation is correct as-is for the documented `application/json` streaming case.
