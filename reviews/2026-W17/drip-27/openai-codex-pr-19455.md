# openai/codex PR #19455 — Add gRPC feedback log sink

- **Repo:** openai/codex
- **PR:** [#19455](https://github.com/openai/codex/pull/19455)
- **Head SHA:** `40c5b5054da9590cc0d01cbeff82d216cb130dff`
- **Author:** rasmusrygaard
- **Size:** +1121 / −0 across 9 files (stacked on #19234)
- **Reviewer:** Bojun (drip-27)

## Summary

Adds a new workspace crate `codex-feedback-log-sink` that implements the
`LogWriter` boundary introduced in PR #19234 over a unary gRPC RPC.
Concretely:

- New crate at `codex-rs/feedback-log-sink/` with `Cargo.toml`, Bazel
  `BUILD.bazel`, generated proto + a regen script.
- Proto contract `codex.feedback_log_sink.v1.proto` defines a single
  unary RPC `AppendLogBatch(AppendLogBatchRequest) -> AppendLogBatchResponse`.
- `src/lib.rs` (664 lines) implements `GrpcFeedbackLogSinkLayer`, which
  doubles as a `tracing_subscriber::Layer` and a `LogWriter`. Owns its
  own bounded `mpsc` queue and worker task; failed batches are dropped
  rather than retried, distinct from the SQLite sink's semantics.

## Key changes

### `codex-rs/Cargo.toml` + `Cargo.lock` (workspace member)

New crate added at line 17 of `codex-rs/Cargo.toml`. Lock file picks up
`codex-feedback-log-sink 0.0.0` with deps on `codex-state`, `prost
0.14.3`, `tonic`, `tonic-prost`, `tracing-subscriber`, `uuid`, plus
dev-deps for `tokio-stream` and `tonic-prost-build`. Versions match what
the rest of the workspace already pins, no new transitive surface.

### `feedback-log-sink/src/proto/codex.feedback_log_sink.v1.proto`

28-line proto defining `FeedbackLogEntry` and `AppendLogBatchRequest /
Response`. Single unary RPC, no streaming. Aligned with the PR body:
"best-effort unary RPC attempts with timeouts, failed batches dropped".

### `feedback-log-sink/scripts/generate-proto.sh`

Regenerates `codex.feedback_log_sink.v1.rs` via the bundled
`generate-proto` example, then post-processes the file:
1. Inserts `#![allow(clippy::trivially_copy_pass_by_ref)]` at line 2 if
   missing.
2. Runs `rustfmt --edition 2024`.
3. Awk pass adds a blank line after the allow attribute for cosmetic
   consistency.

The script is idempotent (the `if ! sed -n '2p' ... | grep -q ...`
guard) which is the right shape for a regen tool that may run on an
already-processed file.

### `feedback-log-sink/src/lib.rs:201–233` — config

```rust
const DEFAULT_RPC_TIMEOUT: Duration = Duration::from_secs(2);

pub struct GrpcFeedbackLogSinkConfig {
    pub endpoint: String,
    pub queue: LogSinkQueueConfig,
    pub rpc_timeout: Duration,
    pub source_process_uuid: Option<String>,
}
```

`normalized()` clamps a zero `rpc_timeout` back to the default. Sensible
default — gRPC timeout 0 is "no deadline", which would let a stalled
backend hold a worker permanently.

### `feedback-log-sink/src/lib.rs:235–290` — layer

`GrpcFeedbackLogSinkLayer::start(config)` constructs the `Endpoint` with
`connect_timeout(config.rpc_timeout).timeout(config.rpc_timeout)`, then
spawns a worker fed by a bounded `mpsc::Sender<RemoteLogCommand>`. The
layer holds only the sender + a `process_uuid`, so the public surface
is `Clone + Send + Sync` and cheap to install on multiple subscribers.

The worker batches accepted entries by count/time and ships them as
`AppendLogBatch`. Per the PR body, failed batches are dropped — no
durable retry buffer on the Rust side. That's the right tradeoff for
a telemetry path where backpressure into the app would be worse than
data loss.

## What's good

- Clean separation: this PR is the *transport*, the boundary
  (`LogWriter`) and the SQLite implementation already landed in #19234
  and #19234 respectively. New code doesn't touch any existing call
  site — it just adds a new `Layer` impl that consumers can opt in to.
- Owning the queue + worker locally instead of resurrecting a shared
  `BufferedSink` abstraction is the right call. The PR body explicitly
  calls this out and the implementation matches.
- Proto file is *checked in alongside* the generated `.rs`, with a
  regen script. That's the easiest mode for reviewers (you can read
  the proto without checking out tonic-prost-build) and keeps the
  build hermetic.
- `DEFAULT_RPC_TIMEOUT = 2s` and the zero-clamp in `normalized()`
  prevent the "infinite timeout via misconfiguration" footgun.
- Bazel `BUILD.bazel` is the minimal `codex_rust_crate` shim
  consistent with how every other crate in the workspace registers.

## Concerns

1. The post-process awk step in `generate-proto.sh:156–159` is fragile
   — it pattern-matches on `previous ~ /clippy::trivially_copy_pass_by_ref/`
   and inserts a blank line if `NR == 3` is non-empty. If `rustfmt`
   ever changes its output to already have that blank line, this awk
   becomes a no-op (fine) but adds confusion. A leading comment in
   the script explaining the intent ("ensure the allow attribute is
   followed by a blank line, regardless of rustfmt version") would
   help the next maintainer.

2. "Failed batches are dropped" is the documented contract, but I
   didn't see a metric/log on drop count exposed back to callers. If
   a backend is silently rejecting every batch, the only signal will
   be tracing logs from inside the worker. For a feedback-log path
   that may be acceptable; worth a follow-up to surface a
   `dropped_batches` counter through the same `LogSinkQueueConfig`
   surface that SQLite uses.

3. `GrpcFeedbackLogSinkLayer::start` is fallible
   (`Result<Self, tonic::transport::Error>`) but only because of the
   endpoint parse — actual connect happens lazily inside the worker.
   That's fine, but means a typo in the endpoint URL fails fast at
   `start` time while a wrong host fails silently later. Worth a
   doc-comment on `start` explicitly noting "constructor only
   validates URL syntax; connectivity errors surface later in the
   worker as dropped batches".

4. The crate exposes `pub mod proto` which re-exports the entire
   tonic-generated client. If consumers import from `proto::` directly
   they'll be coupled to internal generated names that the regen
   script can churn. Worth narrowing the `pub mod proto` to a private
   `mod proto;` and re-exporting only the `FeedbackLogSinkClient`
   alias the layer actually needs externally.

5. No integration test against a real `tonic::transport::Server` —
   the PR mentions a "fake-server test" in the testing section. That's
   typically enough for a unary RPC of this shape, but worth confirming
   the test exercises the *batching* boundary (count threshold + time
   threshold), not just "one event in, one batch out".

## Risk

Low for the existing codebase: zero call-site changes, new crate is
opt-in. Risk is concentrated in whoever wires this layer up next —
they'll need to think about what happens when the remote sink is down
during process startup (hopefully nothing, since the layer's queue is
bounded and full-queue writes drop rather than block).

## Verdict

**merge-after-nits**

The architecture is right, the scope is right, and the failure semantics
are well-thought-out. Concerns 1, 3, and 4 are documentation/visibility
fixes that would make the next consumer's job easier without changing
behaviour. Concern 2 (drop-count metric) is the only one with real
post-merge cost — would block on at least a `tracing::warn!` rate-limited
log of cumulative drops if the existing test doesn't already verify it.

## What I learned

The dual-trait pattern (`tracing_subscriber::Layer` + `LogWriter` on the
same struct) is a clean way to let a single component plug into both
the synchronous tracing fan-out and the async batched-writer interface
without a second adapter type. Worth borrowing for any future remote
sinks (Datadog, OTLP) — the boundary the SQLite sink defined turns out
to be the right shape for gRPC too, which is a good sign that #19234's
abstraction landed in the right place.
