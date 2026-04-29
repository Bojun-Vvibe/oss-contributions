# sst/opencode#24955 — fix(mcp): detect transport close and update status to failed

- **Repo**: sst/opencode
- **PR**: [#24955](https://github.com/sst/opencode/pull/24955)
- **Head SHA**: `5c8a76a1114fa38a745e5e95b3efff6f4bf92aec`
- **Author**: herjarsa
- **Diff stats**: +13 / −0 (1 file)

## What it does

Previously when an MCP server's transport died (process crash, stdio EOF,
SSE socket close), the opencode side kept it marked `status: "connected"`
in `state.status` and the stale `MCPClient` entry stayed in `state.clients`.
Subsequent tool calls would route to a dead client and surface as opaque
errors. Fix installs a `client.onclose` hook in the existing `watch()` setup
that flips status to `"failed"`, drops the entry from `defs`/`clients`, and
closes the now-zombie client handle.

## Code observations

- `packages/opencode/src/mcp/index.ts:475-481` — new `dropConnection(s,
  name, client, reason)` helper. The first guard
  `if (s.clients[name] !== client || s.status[name]?.status !== "connected")
  return` is the load-bearing correctness check: it prevents a late `onclose`
  from a *previous* incarnation of the named server (after a reconnect /
  re-add) from clobbering the current healthy client. Identity check on the
  client reference is the right shape — name alone would be unsafe.
- `:483` — uses `log.warn` rather than `log.error`. Correct call: a
  transport close is expected lifecycle in many setups (user kills the MCP
  process, SSE server scales down) and shouldn't page anyone.
- `:486-487` — both `defs[name]` and `clients[name]` are deleted. Good. If
  only `clients` were dropped, the tool list would still appear in
  introspection until next `ToolListChanged` sweep, leading to "tool exists
  but invocation 404s" confusion.
- `:488` — `void client.close()` is fire-and-forget. That's right for an
  already-dead transport, but if the underlying SDK throws synchronously
  here (e.g. double-close), the error propagates to the bridge. Worth
  wrapping in `try { void client.close() } catch {}` defensively, or relying
  on the SDK's idempotence guarantee — confirm which.
- `:496-498` — `client.onclose = () => bridge.fork(Effect.sync(() =>
  dropConnection(...)))`. Going through `bridge.fork` so the state mutation
  runs in the Effect runtime (not a raw timer callback) is the right
  discipline for this codebase. Reason string `"MCP transport closed
  unexpectedly"` is fine but could be more specific if the SDK passes a
  cause to the close handler — check the
  `@modelcontextprotocol/sdk` `Client.onclose` signature.

## Verdict: `merge-after-nits`

Correct, minimal, addresses a real silent-failure mode. Nits:

1. Wrap `client.close()` at `:488` in a `try { ... } catch (err) { log.debug
   ... }` since the post-onclose state of the SDK client is not contractually
   guaranteed to be re-close-safe across all transport implementations
   (stdio vs SSE vs WS).
2. If the SDK passes an error/code to `onclose`, plumb it through to
   `dropConnection`'s `reason` parameter so the failure log distinguishes
   "stdio EOF" from "WebSocket 1006" from "SSE 502".
3. Add a one-line test (or fixture-level smoke test) using the SDK's
   in-memory transport pair: open, drop the server side, assert that within
   one tick `state.status[name].status === "failed"` and the client is gone
   from `state.clients`. Without this, the next refactor of `watch()` could
   drop the `onclose` wiring and the regression would be invisible until
   production.
4. Consider whether a follow-up reconnect loop belongs here — current
   behavior is "fail and stay failed". For long-lived sessions a
   bounded-retry policy (with status oscillating
   `connected → failed → reconnecting → connected`) would be friendlier
   than forcing the user to remove/re-add the server.

## Follow-ups

- Audit other transport-loss paths: stdio SIGPIPE on `client.notification`,
  HTTP-stream EOF mid-tool-call. Are those routed through `onclose`, or do
  they leak as Promise rejections?
- Surface `status: "failed"` and the reason string in the
  `/mcp` introspection command output and the TUI status bar so users can
  self-diagnose without reading logs.
