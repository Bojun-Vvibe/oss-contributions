# sst/opencode #24932 — fix(mcp): detect transport close and update status to failed

- **PR:** https://github.com/sst/opencode/pull/24932
- **Head SHA:** 5c8a76a1114fa38a745e5e95b3efff6f4bf92aec
- **Files changed:** 1 file, +13 / −0 (`packages/opencode/src/mcp/index.ts`)
- **Verdict:** `merge-after-nits`

## What it does

Adds a transport-close handler to the MCP client lifecycle. Previously, when an MCP
server's stdio/transport died after a successful connect, opencode kept the cached
client and its tool definitions in `s.clients[name]` / `s.defs[name]` indefinitely, so
subsequent tool invocations would fail at the protocol level with no surfaced "server
went away" signal. The new `dropConnection` helper (`mcp/index.ts:475-482`) flips the
status entry to `{ status: "failed", error: reason }`, deletes the client + defs, and
calls `client.close()`. The hook is wired in `watch()` at `mcp/index.ts:497-499` via
`client.onclose = () => bridge.fork(Effect.sync(() => dropConnection(...)))`.

## What's good

- The identity check `s.clients[name] !== client` (line 476) correctly guards against a
  reconnect-then-old-client-fires-onclose race — only the *current* client for a name
  can transition the slot to failed. That's the right shape.
- Routes through `bridge.fork(Effect.sync(...))` rather than mutating state from a raw
  callback, keeping the Effect runtime as the single state owner.

## Nits / risks

1. The reason string is a static `"MCP transport closed unexpectedly"` even on clean
   shutdown paths — if `mcp.shutdown()` itself triggers `onclose`, every clean teardown
   will log a `log.warn` and mark the server failed. Worth gating on a "we're shutting
   down" flag or skipping the warn when the layer is releasing.
2. `void client.close()` after the transport already closed is harmless but produces a
   second close-error in some SDKs; consider try/catching.
3. No test added. A unit test that fakes `client.onclose()` and asserts
   `s.status[name].status === "failed"` plus def cleanup would pin the contract.

## What I learned

The `s.clients[name] !== client` identity guard is a generalizable pattern for any
"async callback might fire after the slot was replaced" scenario — cheaper than
generation counters and no allocation.
