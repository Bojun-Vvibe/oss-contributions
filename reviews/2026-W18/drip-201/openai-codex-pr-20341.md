# openai/codex #20341 — app-server: switch remote control to protocol v3 segmentation

- **Author:** euroelessar (Ruslan Nigmatullin)
- **SHA:** `848076a`
- **State:** OPEN
- **Size:** +894 / -29 across 7 files in `codex-rs/app-server/src/transport/remote_control/`
- **Verdict:** `merge-after-nits`

## Summary

Lands a wire-format upgrade for the app-server remote-control channel: introduces
`ClientMessageChunk` / `ServerMessageChunk` variants on `protocol.rs:50-57,98-103`
each carrying `(segment_id, segment_count, message_size_bytes, message_chunk_base64)`
and rewrites the simple `Ack` variant into `Ack { segment_id: Option<usize> }` at
`protocol.rs:54-63` so per-chunk acks can survive a reconnect. The bulk of the new
code lives in two added files: `segment.rs` (+414 lines, the chunk
splitter/reassembler) and `segment_tests.rs` (+235 lines, the regression suite). The
`websocket.rs` rewrite (+213 / -25) threads the segmenter through the send/receive
paths and adds chunk-aware retention bookkeeping.

## Reasoning

The protocol shape is right — moving from "one message = one envelope" to "one
message = N chunks each independently ackable" is the correct fix for the
WebSocket-frame-size-cap problem that's been showing up in long tool-output streams,
and the per-chunk `segment_id` on the ack means a reconnect doesn't have to retransmit
the entire message. The `Ack { #[serde(skip_serializing_if = "Option::is_none")]
segment_id: Option<usize> }` shape at `protocol.rs:60-62` preserves wire compatibility
with v2 clients that send bare `Ack` (deserializes as `segment_id: None`), which is
the right migration discipline. The 235-line `segment_tests.rs` is healthy coverage
for a new wire format. Three nits before merge: (1) `ClientEvent::Ack { .. }` at
`client_tracker.rs:198` swallows the new `segment_id` rather than threading it
through to the per-stream cursor logic — if the goal is per-chunk retention, the
tracker needs to learn the new field, otherwise the chunk-aware retention bookkeeping
in `websocket.rs` is the only place doing it and the tracker remains envelope-scoped;
(2) `message_size_bytes: usize` is duplicated on every chunk of the same message and
the receiver could verify on reassembly that all chunks agree — a one-line `assert_eq!`
in the reassembler would catch an off-by-one segmenter bug fast; (3) base64-encoding
the chunk body inflates the wire by ~33% which is a real cost on the message-size
ceiling this PR is trying to push out — worth a comment explaining why base64 was
chosen over a binary WebSocket frame (likely "stays JSON-clean for the existing
JSONRPCMessage envelope" but the rationale should be visible). Test files are not
runtime code so the +414 / +235 line additions don't bloat the binary.
