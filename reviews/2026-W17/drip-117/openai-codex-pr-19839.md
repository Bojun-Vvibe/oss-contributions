# PR #19839 — Trace cancelled inference streams

- **Repo**: openai/codex
- **PR**: #19839
- **Head SHA**: `79391f0f965d790b3caeec0b941305bef1b5d23d`
- **Author**: cassirer-openai
- **Size**: +383 / -33 across 10 files
- **Verdict**: **merge-as-is**

## Summary

Closes the rollout-trace-completeness gap where the inference
node was left in `Running` whenever the consumer dropped the
`ResponseStream` before the provider sent its terminal
`response.completed` event. The fix wires a `CancellationToken`
between the `ResponseStream` and the mapper task that's draining
the upstream `api_stream`, with a `Drop` impl on
`ResponseStream` that fires `consumer_dropped.cancel()`. The
mapper's main loop becomes a `tokio::select!` that races the
upstream event against the cancel signal, recording either
`record_cancelled(reason, &items_added)` (drop case) or
`record_failed("stream closed before response.completed", &items_added)`
(natural EOF without terminal event) instead of silently
returning.

## Specific changes

- `codex-rs/core/src/client.rs:1635-1739` — replaces
  `while let Some(event) = api_stream.next().await` with a
  `loop { tokio::select! { _ = consumer_dropped.cancelled() =>
  ...; event = api_stream.next() => event } }`. On the cancel
  branch, calls `inference_trace_attempt.record_cancelled("response
  stream dropped before provider terminal event", &items_added)`
  and returns. After the loop exits naturally without ever seeing
  `Completed`, calls `record_failed("stream closed before
  response.completed", &items_added)` so the trace doesn't strand
  in `Running`.
- `codex-rs/core/src/client_common.rs:175-196` — `ResponseStream`
  gains a `pub(crate) consumer_dropped: CancellationToken` field
  and an `impl Drop` that calls `self.consumer_dropped.cancel()`
  on drop. The `Stream` impl is unchanged — just polls
  `rx_event` — so cancellation is purely a side channel for the
  mapper, not the stream consumer.
- `codex-rs/core/src/client.rs:1233-1376` — `record_failed`
  signature changes from `record_failed(&err)` to
  `record_failed(&err, &[])` at the three pre-stream call sites
  (unauthorized retry, post-handle-unauthorized failure, mapper
  request-build failure) so they pass an empty `items_added`
  slice for trace consistency. Backwards-compatible at the call
  sites — they didn't have items to report and now explicitly
  say so.
- `codex-rs/rollout-trace/src/inference.rs` (+83/-26) and
  `reducer/inference.rs` (+33) thread the
  `record_cancelled(reason, items_added)` through the
  trace-event payload (new variant in `raw_event.rs:+11`), and
  the reducer collapses Cancelled-with-partial-items into a
  terminal `Cancelled` execution status that preserves the
  observed output items rather than dropping them.
- `codex-rs/rollout-trace/src/reducer/inference_tests.rs` (+92)
  pins the reducer behavior across three cases: cancelled with
  zero items, cancelled with partial items, and failed-without-terminal
  with partial items. `client_tests.rs:163-260`
  (`dropped_response_stream_traces_cancelled_partial_output`,
  +103) is the integration test — builds a real `TraceWriter`
  + `InferenceTraceContext`, starts an attempt, drops the
  `ResponseStream` mid-flight, then `replay_bundle`s the trace
  and asserts (a) the inference node terminated as `Cancelled`,
  (b) the cancellation reason string is preserved, and (c) the
  items observed before the drop are present in the replayed
  output. That's the right shape — assert against the replay
  surface, not the writer's intermediate state.
- `codex-rs/rollout-trace/src/reducer/thread.rs` (+6) /
  `reducer/mod.rs` (+14) — closes still-running inference nodes
  when their owning turn ends, so a reduced trace can never
  contain a stale `Running` node hanging off a completed turn.

## Risks

1. **Drop ordering on shutdown.** If the runtime is shutting down
   and the mapper task is cancelled by the runtime *before* the
   `ResponseStream` is dropped on the consumer side, the cancel
   signal still fires (since `Drop` runs synchronously) but the
   mapper is already gone, so the `record_cancelled` call inside
   the mapper never executes. Net behavior in that case is the
   same as the old behavior (no terminal event recorded), which
   is no worse than today — but a follow-up that records the
   "abandoned" status from the `ResponseStream::drop` side itself,
   not the mapper, would close the last gap.
2. **`CancellationToken` is single-shot.** A second call to
   `cancel()` is a no-op, which is fine here, but it means the
   reason string (`"response stream dropped before provider
   terminal event"`) is hardcoded in the mapper. If callers ever
   want to thread a *richer* cancel reason from the consumer
   site (e.g. "user pressed Ctrl-C" vs "session switched"), the
   `CancellationToken` will need to be paired with a
   `OnceCell<String>` or similar.
3. **The natural-EOF `record_failed("stream closed before
   response.completed", ...)` line at the end of the mapper
   classifies provider EOF-without-terminal as a failure.** That's
   the right call for tracing (otherwise the node strands), but
   it's worth confirming the upstream telemetry / alerting
   doesn't page on every benign disconnect — the message string
   is distinct enough to filter, but a "graceful_close=true"
   side-flag on the failure record would let downstream consumers
   distinguish protocol-error from clean-EOF.

## Verdict

`merge-as-is` — the architecture (CancellationToken side-channel
+ Drop-fires-cancel + select-loop-in-mapper) is the right shape
for a partial-output-preserving cancel path, the
record_failed-with-items-vec signature change is consistently
applied across all three pre-stream call sites, and the test
coverage hits all three terminal cases (cancelled with items,
cancelled without, failed-without-terminal with items) at both
the reducer and integration levels.

## What I learned

When an async producer-consumer split shares a "this stream is
done" notion, the two sides need *both* directions of signaling:
producer-tells-consumer (the existing `rx_event` channel) and
consumer-tells-producer (the new `CancellationToken`). The bug
shape — strand in `Running` when only the consumer-tells-producer
direction is missing — is a classic asymmetric-signaling failure,
and the fix is the textbook addition: one-shot signal token with
a `Drop` hook on the consumer side, racy `select!` on the
producer side. The bonus invariant from `reducer/mod.rs` ("close
still-running inference nodes when their owning turn ends") is
the cleanup belt that catches whatever still slips through.
