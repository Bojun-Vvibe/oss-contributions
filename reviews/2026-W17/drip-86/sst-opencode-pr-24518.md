# Review — sst/opencode#24518: feat(httpapi): bridge event stream

- **Repo:** sst/opencode
- **PR:** [#24518](https://github.com/sst/opencode/pull/24518)
- **Head SHA:** `2b98b9a6f4bc5641fcd62baaa8ee5af95e038f0d`
- **Author:** sst maintainer / `kit/httpapi-event-stream` branch
- **Size:** +120 / -2 across 5 files
- **Verdict:** `merge-after-nits`

## Summary

This PR adds the SSE `GET /event` route to the new raw-Effect HTTP surface (gated by `OPENCODE_EXPERIMENTAL_HTTPAPI`), checking off the last "bridge parity" tracker entry in `packages/opencode/specs/effect/http-api.md:187` (`event` row flips from `special` to `bridged`) and `:319` (`- [x] GET /event`). The Hono legacy route stays as default; the new route lives at `packages/opencode/src/server/routes/instance/httpapi/event.ts:1-62` and is wired into the experimental router via `httpapi/server.ts:13,67` and `httpapi/index.ts:18`. Test coverage lands at `test/server/httpapi-event.test.ts` (+52).

## Technical assessment

The route shape is correct and idiomatic. The handler uses `Stream.callback<string>` to bridge `Bus.subscribeAllCallback` → an Effect Queue, with three lifecycle correctness wins: (1) immediate `server.connected` push at `event.ts:30` so clients can detect connection without waiting for the first bus event, (2) a `setInterval` heartbeat at `:33-35` every 10 s emitting `server.heartbeat` to defeat intermediary idle-disconnect (correct interval — under most LB defaults of 60 s but not so frequent it adds noise), and (3) a re-entrant-safe `stop()` at `:18-25` guarded by a `done` flag that clears the heartbeat, unsubscribes, and ends the queue. The `Effect.addFinalizer(() => Effect.sync(stop))` at `:43` correctly hooks the scope so client disconnect tears everything down. The `InstanceDisposed` short-circuit at `:39` is a nice touch — server shutdown actively closes the stream rather than waiting for a heartbeat write to fail.

The response framing at `:46-56` is solid: `text/event-stream`, `Cache-Control: no-cache, no-transform`, `X-Accel-Buffering: no` (the canonical nginx-buffering opt-out), and `X-Content-Type-Options: nosniff`. Encoding via `Stream.encodeText` at `:45` is the right primitive for SSE byte framing.

## Nits worth addressing pre-merge

1. **Heartbeat-during-shutdown race.** The `setInterval` at `:33` is captured by the closure at `:21` (`clearInterval(heartbeat)`) but `heartbeat` is declared *after* `stop()` is defined — this works under JS hoisting because `stop()` only reads `heartbeat` when called, but if `bus.subscribeAllCallback` fires `InstanceDisposed` synchronously (even rare), `stop()` would see `heartbeat === undefined` and `clearInterval(undefined)` is a no-op silently leaking the timer. Move `const heartbeat = setInterval(...)` above the `stop` declaration, or initialize as `let heartbeat: ReturnType<typeof setInterval> | null = null` and null-check in `stop()`.

2. **Subscribe-before-push ordering.** The `push({ type: "server.connected" })` at `:30` happens *before* `unsubscribe = yield* bus.subscribeAllCallback(...)` at `:37`. There's a tiny race where a bus event published between `push("connected")` and `subscribeAllCallback` resolving will be missed by this client. The fix is to subscribe first, then push connected — same end-user observable order on the wire because the queue is FIFO.

3. **Test coverage.** `test/server/httpapi-event.test.ts` (52 lines added) was not visible in the diff snippet but the PR body lists it being run. Confirm it asserts: (a) `server.connected` arrives first, (b) heartbeat fires within 11 s under `vi.useFakeTimers()`, (c) `InstanceDisposed` closes the stream and the response future resolves, (d) abrupt client disconnect runs `stop()` and unsubscribes from the bus (no-leak assertion via spy).

4. **No flag-disabled regression test.** The PR preserves the legacy Hono route as the non-flag default, but there's no test asserting the Hono route still serves `/event` when `OPENCODE_EXPERIMENTAL_HTTPAPI` is unset. A one-line "without the flag, route is Hono" smoke would prevent a future cleanup PR from accidentally deleting the legacy path before bridge parity ships GA.

## Verdict rationale

`merge-after-nits` because the code is correct and finishes a tracker line. The two ordering concerns (heartbeat declaration, subscribe-before-push) are micro-bugs that would be hard to repro but easy to fix; the test-coverage asks lock the contract for the bridge work to come.
