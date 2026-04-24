# openai/codex#19246 — increase app-server WebSocket outbound buffer

**What changed.** Adds a WebSocket-only outbound writer queue capacity of `32 * 1024` (the PR description says `64 * 1024` but the code lands `32 * 1024`), replacing the shared `CHANNEL_CAPACITY = 128` only for the WS data writer. A `const _: () = assert!(WEBSOCKET_OUTBOUND_CHANNEL_CAPACITY > CHANNEL_CAPACITY)` guards the invariant. Internal/control channels and the existing disconnect-on-full behavior are untouched.

**Why it matters.** Remote TUI clients streaming a healthy turn could fill the 128-deep mpsc queue during normal output bursts and get disconnected by the existing "channel full = drop client" policy. This is a small, surgical mitigation versus the more invasive overflow/backpressure pipeline in #18265.

**Concerns.**
1. **PR description / code mismatch.** Description says "64 * 1024" — code says `32 * 1024`. Pick one; reviewers should confirm whichever is intended is what's tested.
2. **Memory ceiling per connection is now ~250x larger.** With `QueuedOutgoingMessage` carrying serialized JSON-RPC frames, a stuck-but-not-dead client can buffer tens of MB before disconnect. On a server with N concurrent stuck-ish connections, this is a real footprint. Worth either documenting the worst-case bytes-per-connection or capping by bytes rather than message count.
3. **Disconnect-on-full is now much rarer but not gone.** The actual failure mode shifts from "fast client gets killed during a burst" to "slow-but-alive client builds up multi-second backlog before getting killed." The latter looks like UI lag, not a disconnect, which is a worse UX. A bytes-budget or time-budget would be a better long-term fix; this PR should land with a follow-up issue filed.
4. **Test coverage gap.** `broadcast_does_not_block_on_slow_connection` exercises the non-blocking router, not the new headroom. There's no regression test that asserts a 200-message burst no longer disconnects a healthy WS client. Worth adding.
5. **Const assertion is good defensive style** — keep it.

Net: pragmatic short-term fix. Land it but file the bytes-budget follow-up.
