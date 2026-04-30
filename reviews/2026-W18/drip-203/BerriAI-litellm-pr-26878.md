# BerriAI/litellm #26878 — fix(guardrails): preserve responses event streams in presidio output masking

- **Author:** Sameerlite
- **SHA:** `5622dd2`
- **State:** OPEN
- **Size:** +58 / -2 across 2 files
  (`litellm/proxy/guardrails/guardrail_hooks/presidio.py`,
  `tests/test_litellm/proxy/guardrails/guardrail_hooks/test_presidio.py`)
- **Verdict:** `merge-after-nits`

## Summary

Closes a real production bug where Presidio's `apply_to_output` streaming
hook silently dropped any chunk that was neither `ModelResponseStream` nor
`bytes`. For `/v1/responses`-shaped event streams (which yield typed event
objects like `response.created`, `response.output_text.delta`,
`response.completed`), the entire stream got eaten — including the
load-bearing `response.completed` event — causing downstream callers (the PR
body specifically calls out the SDK that consumes this surface) to disconnect
mid-stream waiting for a lifecycle terminator that never arrives. Fix at
`presidio.py:1163,1170-1182` adds an `else` arm that flushes any buffered
`ModelResponseStream` chunks to the consumer, sets a
`passthrough_due_to_unknown_stream_shape = True` latch, and yields the
unknown-shape chunk transparently; subsequent `ModelResponseStream` chunks
are then yielded directly rather than buffered for masking. Locked by a new
regression `test_apply_to_output_streaming_unknown_events_passthrough` at
`test_presidio.py:2122-2161` exercising the three-event lifecycle and
asserting both `received == events` (object identity) and the type-string
ordering.

## Reasoning

Three things this PR gets right:

1. **The latch-and-flush shape is the correct recovery semantics.** Once an
   unknown-shape chunk arrives, two things are simultaneously true: (a) the
   already-buffered `ModelResponseStream` chunks must still reach the
   consumer in order, and (b) the masking pass cannot be applied because
   the stream isn't a single homogeneous shape anymore. The fix correctly
   handles both — flushes the buffer in original order, then switches to
   transparent passthrough for the rest of the stream — instead of either
   dropping the buffered chunks (silent corruption) or trying to mask a
   mixed-shape stream (would crash on the next event-shape chunk).

2. **`if passthrough_due_to_unknown_stream_shape: return` after the stream
   ends at `:1182-1183` is the correct early-exit.** Without it, the
   downstream "if not all_chunks: warn(...)" branch at `:1184-1188` would
   fire spuriously every time the stream entered passthrough mode (because
   `all_chunks` was reset to `[]` on the latch transition), producing a
   confusing warning log for what is now intended behavior.

3. **The regression test asserts object identity (`received == events` with
   the events list directly), not structural equality.** `FakeResponsesEvent`
   doesn't define `__eq__`, so this is identity comparison via Python's
   default `object.__eq__`, which means a future regression that
   reconstructs equivalent-but-different event objects would fail this
   assertion. That's the right strength for a "preserve the wire stream"
   contract.

Two nits worth a follow-up before merge:

1. **The fix recovers from the unknown-shape transition but silently
   stops masking the previously-buffered `ModelResponseStream` chunks.**
   `all_chunks` is flushed unmasked at `:1175-1177`. That's the correct
   safety choice for the responses-stream case (where masking the buffered
   chunks separately and then yielding raw events would produce an
   inconsistent stream), but a one-line `verbose_proxy_logger.info` at the
   transition point ("presidio: unknown stream shape detected at chunk N,
   switching to transparent passthrough; {len(all_chunks)} prior chunks
   yielded unmasked") would let operators detect when a guardrail policy
   they think is enforcing PII masking is actually being bypassed by the
   stream-shape detector.

2. **`passthrough_due_to_unknown_stream_shape` is a 41-character local
   that appears 4 times in the diff hunk.** The name is correctly precise
   for the comment-as-name purpose, but a refactor to a small helper class
   (`MaskingState.Buffering` / `MaskingState.Passthrough`) would express
   the same invariant with fewer lines and prevent a future contributor
   from forgetting to set the latch in a third arm. Not a blocker, just a
   readability/safety nudge for the next iteration.

The fix correctly diagnoses (Codex SDK stream disconnects on missing
`response.completed`) and ships with a regression test that pins the
preserve-order-and-identity contract. Worth the merge-after-nits if the
maintainer adds the operator-visibility log; otherwise straight merge-as-is
is also defensible because the silent-bypass risk only applies to mixed
streams which were already broken before this PR.
