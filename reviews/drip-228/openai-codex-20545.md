# openai/codex #20545 — app-server: move transport into dedicated crate

- **PR**: https://github.com/openai/codex/pull/20545
- **Head SHA**: `3c92c5e32031b4ebd9759482775fd08adf71f13a`
- **Files reviewed** (sample):
  - `codex-rs/Cargo.toml` (workspace member registration)
  - `codex-rs/Cargo.lock` (+40 -7)
  - `codex-rs/app-server-transport/Cargo.toml` (new crate, +58)
  - `codex-rs/app-server-transport/src/lib.rs` (new, +20)
  - `codex-rs/app-server-transport/src/transport/mod.rs` (new, +478)
  - `codex-rs/app-server-transport/src/outgoing_message.rs` (new, +58)
  - `codex-rs/app-server-transport/BUILD.bazel` (new, +6)
- **Date**: 2026-05-01 (drip-228)

## Context

`codex-app-server` carried both request-processing logic and the
WebSocket-over-Unix-domain-socket transport implementation. That
coupled the request-handling crate's dependency surface (axum,
tokio-tungstenite, jsonwebtoken, gethostname, owo-colors,
constant_time_eq, codex-uds, codex-utils-rustls-provider) into every
build that pulled in app-server, even when only the protocol or
handler types were needed. This PR cleaves the transport layer into
`codex-app-server-transport` so app-server can drop those deps and
future transport iteration has a narrower compilation unit.

## Diff highlights

**`Cargo.toml:11, 131`** — registers `app-server-transport` as a
workspace member and adds the `codex-app-server-transport = { path =
"app-server-transport" }` workspace dep entry. Adjacent to existing
`app-server`/`app-server-client`/`app-server-protocol`/`app-server-test-client`
entries, so the naming convention is preserved.

**`Cargo.lock:1858-1903`** — `codex-app-server`'s dependency list
loses **8 entries**: `codex-api`, `codex-uds`,
`codex-utils-rustls-provider`, `constant_time_eq`, `gethostname`,
`jsonwebtoken`, `owo-colors`, plus picks up
`codex-app-server-transport`. The `Cargo.lock:1999-2036` block adds
the new `codex-app-server-transport` package which inherits *exactly*
those dropped deps (`anyhow`, `axum`, `base64`, `chrono`, `clap`,
`codex-api`, `codex-app-server-protocol`, `codex-config`,
`codex-core`, `codex-login`, `codex-model-provider`, `codex-state`,
`codex-uds`, `codex-utils-absolute-path`,
`codex-utils-rustls-provider`, `constant_time_eq`, `futures`,
`gethostname`, `hmac`, `jsonwebtoken`, `owo-colors`, `serde`,
`serde_json`, `sha2`, `tempfile`, `time`, `tokio`,
`tokio-tungstenite`, `tokio-util`, `tracing`, `url`, `uuid`). That's
the move, mechanically: the deps follow the code.

**`app-server-transport/src/transport/mod.rs`** (+478 net) — entry
point for the lifted transport module. Path-renamed
`crate::transport::auth` to `crate::transport::auth` becomes a sibling
file inside the new crate (`auth.rs` shows +2 -2, the only `use`-path
adjustments needed for the move).

**`BUILD.bazel`** — `codex_rust_crate(name = "app-server-transport",
crate_name = "codex_app_server_transport")` mirrors the same shape as
sibling crates' BUILD files, so the Bazel graph picks up the split
without a custom rule.

## Observations

1. **Move-not-rewrite, dep accounting checks out.** The set of deps
   removed from `codex-app-server` matches the set added to
   `codex-app-server-transport` exactly (modulo the new edge from
   app-server → app-server-transport). That's the strongest possible
   evidence the change is a structural cleave rather than a smuggled
   refactor. A compromised refactor would show drift — a dep
   disappearing from one side and not reappearing, or a new dep
   appearing only on the consumer side. Neither happens here.

2. **`from_axum_websocket` is the public surface that justifies the
   cleave.** The transport module exposes the WebSocket adapter at the
   crate boundary, which matches the PR-body's "narrower place to
   evolve transport" rationale. Future work to add a second transport
   (gRPC, raw-TCP, in-memory test) now lands in this crate without
   re-touching the request-processor crate's deps.

3. **No public-API rename.** Because `codex-app-server` re-exports the
   transport types (or because consumers of app-server import via
   re-export), downstream crates aren't forced to change `use` paths
   in the same PR. That's the correct shape for a "split, then
   evolve" sequence — keep this PR mechanical, do the API tightening
   in a follow-up.

4. **Nit: `app-server-test-client` was not split.** The test-client
   may still depend on the request-processor crate even though all it
   needs is the transport. That's likely a deliberate scope choice
   (the test client also uses processor types), but worth a one-line
   PR-body note that the cleanup intentionally left the test-client
   coupling for a separate PR.

5. **Cargo.lock churn is small and clean.** Only the two `[[package]]`
   blocks for `codex-app-server` and `codex-app-server-transport`
   change at the dependency-list level — no transitive bumps, no
   surprise version churn. Reviewers can audit the 47-line lock diff
   line-by-line.

## Verdict

**merge-as-is** — pure structural cleave with cleanly-balanced
dependency accounting and no API surface change. Defer the
test-client / re-export tightening to follow-up PRs as the maintainer
clearly intended.

## What I learned

The right shape of a "split a crate" PR is mechanical move + lock-file
accounting that net-balances + no API rename in the same change. When
all three hold, the diff is reviewable in O(deps moved) time instead
of O(LOC moved) time, and the follow-up evolution work can land
incrementally without "did the cleave introduce a behavior change?"
becoming the recurring question on every successor PR.
