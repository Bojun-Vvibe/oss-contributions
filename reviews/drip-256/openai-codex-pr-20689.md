# openai/codex PR #20689 — Inject state DB, agent graph store

- URL: https://github.com/openai/codex/pull/20689
- Head SHA: `97ddb4da5ff6c5e16d5d7ceb3d5f93b024d7bd4b`
- Author: rasmusrygaard
- Verdict: **needs-discussion**

## Summary

Adds a `RemoteAgentGraphStore` gRPC client to the `codex-agent-graph-store` crate, behind a new `remote` module, alongside a checked-in protobuf-generated Rust file (`src/remote/proto/codex.agent_graph_store.v1.rs`) and a `scripts/generate-proto.sh` regeneration helper. Wires `tonic`/`tonic-prost`/`prost` 0.14.3 into both `[dependencies]` and `[dev-dependencies]`, registers the crate in the `app-server` Cargo manifest, and exposes `RemoteAgentGraphStore` from the crate root.

## Line-level observations

- `codex-rs/agent-graph-store/Cargo.toml` lines 18–34: `prost = "0.14.3"` is added as a direct dep but pinned only as a string version, while `tonic`/`tonic-prost` go through `workspace = true`. This split is fragile — a future workspace bump of prost will desync this crate. Prefer `prost = { workspace = true }` and let the workspace own the version.
- `codex-rs/agent-graph-store/src/remote/AGENTS.md` (new file): explicitly tells future maintainers "do not add build-time protobuf generation … unless the Bazel/Cargo story is intentionally changed." Good guardrail, but the policy needs to live somewhere CI can enforce — otherwise a regen drift will silently land. Consider a CI job that runs `scripts/generate-proto.sh` and `git diff --exit-code`.
- `codex-rs/agent-graph-store/scripts/generate-proto.sh`: the awk/sed pipeline that prepends `#![allow(clippy::trivially_copy_pass_by_ref)]` is brittle — relies on `sed -n '2p'` containing the exact string. If `tonic-prost-build` changes the file header (e.g. version comment), the check silently no-ops. Suggest a more structural insertion (idempotent block-marker).
- `src/remote/helpers.rs` lines 33–42: `remote_status_to_error` only special-cases `tonic::Code::InvalidArgument`. `NotFound`, `PermissionDenied`, `Unauthenticated`, `DeadlineExceeded`, `Unavailable` all collapse into `Internal { … }`. For a store backing user-visible features, `NotFound` and `Unavailable` deserve their own variants so callers can implement retry/backoff or surface "not found" without a stack trace.
- `src/remote/mod.rs` lines 1–14: the `RemoteAgentGraphStore` only stores `endpoint: String`. Each call presumably establishes a new `AgentGraphStoreClient` connection — confirm there's a connection-pool / reusable channel, or this becomes a per-RPC TCP+TLS handshake.
- The PR registers the crate in two manifests (`app-server`, plus another at line ~2416 of the Cargo.lock-adjacent diff) but only exposes `RemoteAgentGraphStore` — there's no test in the diff that actually constructs one against an in-process gRPC server (the `tonic-prost-build` dev-dep suggests one was intended).

## Suggestions

1. Use `prost = { workspace = true }` and add `prost`/`tonic-prost` to the workspace manifest if not already there.
2. Expand `remote_status_to_error` to map at least `NotFound`, `Unavailable`, `DeadlineExceeded`, `PermissionDenied`/`Unauthenticated` to dedicated variants — `Internal` is a footgun for callers.
3. Add an integration test that spins up a `tonic` test server (the dev-deps already include `tokio-stream` features=["net"] and `tonic` features=["router","transport"]) and exercises at least one round-trip through `RemoteAgentGraphStore`.
4. Wire `scripts/generate-proto.sh` into CI with `git diff --exit-code` so checked-in `.rs` can't drift from the `.proto`.
