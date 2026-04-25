# PR #19465 — Add gRPC feedback log sink

- Repo: openai/codex
- Head SHA: 99bcadd0b13e0ba2bb99c7dde053f97f7770fce9
- Files touched: 9 (+1121 / -0)

## Summary

New `codex-feedback-log-sink` crate that implements the `LogWriter`
boundary carved out by an earlier PR (drip-17 #19234) and pairs it
with a `tracing_subscriber::Layer`. The sink owns its own bounded
mpsc queue + worker, formats accepted tracing events into
`codex_state::LogEntry` via `DefaultFields::format_fields`,
batches by count + time, and ships batches as **best-effort unary**
`AppendLogBatch` RPCs to a remote
`codex.feedback_log_sink.v1.FeedbackLogSink` server. Ships the
`.proto`, the generated `proto/codex.feedback_log_sink.v1.rs`, a
`generate-proto` example + shell script, and a fake-server flush
test.

## Specific findings

- `codex-rs/feedback-log-sink/Cargo.toml:18-32` pulls in `tonic`,
  `tonic-prost`, `prost = "0.14.3"`, `tracing-subscriber`, and
  `uuid` as runtime deps and `tonic-prost-build = "=0.14.3"` only
  as a dev-dep. That's the right split — the generated proto is
  checked in, so `tonic-prost-build` only needs to exist for the
  `examples/generate-proto.rs` regen path.
- `codex-rs/feedback-log-sink/scripts/generate-proto.sh:24-36` does
  two post-passes on the generated file: an unconditional
  `#![allow(clippy::trivially_copy_pass_by_ref)]` insert at line 2
  (idempotent — guarded by a `sed -n '2p' | grep` check), then an
  `awk` blank-line normalisation between the inserted allow and the
  rest of the file. Worth pinning the exact `tonic-prost-build`
  version in the script (it already pins via Cargo) so a regen
  doesn't silently switch lint-suppress shapes.
- `codex-rs/feedback-log-sink/src/lib.rs:1-45` declares the layer
  via `#[path = "proto/codex.feedback_log_sink.v1.rs"] pub mod
  proto`. That's the standard pattern but means the layer module
  re-exports the entire generated client surface — fine for now but
  consider a thin facade if/when consumers grow.
- `codex-rs/feedback-log-sink/src/proto/codex.feedback_log_sink.v1.proto:1-28`
  is a **single unary RPC** (`AppendLogBatch`). The PR body
  explicitly calls out best-effort delivery with batch drops on
  failure rather than retries or durable buffering. That's a
  defensible policy for a feedback sink (you don't want one bad
  endpoint to back-pressure inference) but it should be loud in
  observability — if the worker is dropping batches, an operator
  needs to be able to tell from local metrics. The PR description
  doesn't mention a drop counter; reviewer should confirm one
  exists in the worker loop.
- `codex-rs/Cargo.toml:14-18` registers the new crate. No version
  pin (uses `version.workspace = true`) which is correct.

## Risks / nits

- Best-effort drop on failed `AppendLogBatch` + a bounded queue
  means under sustained backend outage the sink will silently
  shed log volume. Suggest emitting a tracing warn through a
  separate logger (so it doesn't recurse into the same sink) on
  every Nth drop, plus a counter exposed via the existing
  analytics surface.
- The flush barrier (referenced in the PR body and used by the
  fake-server test) needs to be wired by callers at shutdown.
  Confirm the app-server / TUI shutdown paths actually call it,
  otherwise tail logs at process exit will be silently lost — a
  follow-up integration test in the consumer crate would lock
  this down.
- `LogSinkQueueConfig` in `codex_state::log_db` is the shape
  contract here. Without seeing both sides of the boundary in
  this diff, reviewer should sanity-check that the queue config
  defaults align (esp. `max_batch_size` vs the bounded mpsc
  capacity) — a queue capacity smaller than `max_batch_size`
  would deadlock the worker.

## Verdict

**merge-after-nits** — The crate boundary, proto shape, and the
fake-server end-to-end test are clean. Add a drop-counter +
shutdown-flush wiring confirmation before landing, and consider
pinning the post-process script's `tonic-prost-build` to the
exact Cargo version to keep regen reproducible.

## What I learned

Best-effort remote log shipping is the right default for a
feedback sink (you don't want it to back-pressure inference or
amplify a backend outage), but the failure mode — silent batch
drops under sustained outage — only becomes operable if you
expose a drop counter on the worker side. Otherwise observability
people will keep asking why the central sink looks under-fed and
have no way to localize the loss to a specific pod.
