---
pr: 19575
repo: openai/codex
sha: 859ed739c0f65f1a6cfc8348c79f13079c8934f1
verdict: merge-after-nits
date: 2026-04-26
---

# openai/codex #19575 — Add cloud executor registration to exec-server

- **URL**: https://github.com/openai/codex/pull/19575
- **Author**: miz-openai (Michael Zeng)
- **Head SHA**: 859ed739c0f65f1a6cfc8348c79f13079c8934f1
- **Size**: +476/-3 across 8 files (`codex-rs/cli/src/main.rs`, `codex-rs/exec-server/Cargo.toml`, `codex-rs/exec-server/README.md`, `codex-rs/exec-server/src/{client.rs,cloud.rs,lib.rs}`, `Cargo.lock`)

## Scope

First milestone for remote-exec end-to-end. Adds a new `--cloud` mode to `codex exec-server` that:

1. Loads ChatGPT auth from normal Codex config via `AuthManager::shared_from_config(&config, /*enable_codex_api_key_env*/ false)` (`cli/src/main.rs:1280`).
2. Calls `POST /api/cloud/executor` on the cloud-environments service to register an executor.
3. Receives a signed rendezvous websocket URL.
4. Connects to that websocket and serves the existing JSON-RPC `ExecServerHandler` over it via `ConnectionProcessor::run_connection(JsonRpcConnection::from_websocket(...))` (`cloud.rs:339`).
5. Reconnects with exponential backoff (1s → 30s cap) on disconnect (`cloud.rs:344-346`).

New CLI flags: `--cloud`, `--cloud-base-url`, `--cloud-environment-id`, `--cloud-name` (`cli/src/main.rs:454-468`). `CODEX_CLOUD_ENVIRONMENTS_BASE_URL` env var fallback for the base URL (`cloud.rs:218`).

## Specific findings

- **Auth retry envelope is right-shaped.** `post_json` at `cloud.rs:248-280` loops at most twice: on a `401`/`403` it calls `recover_unauthorized(&self.auth_manager).await` once and retries. The `unreachable!` at line 281 is properly guarded by the bounded `for attempt in 0..=1` — clippy won't flag it. The bound choice (single retry) is appropriate: any auth failure beyond one refresh is operator error, not transient.

- **Idempotency-id construction is good and slightly subtle.** `default_idempotency_id` at `cloud.rs:166-179` SHA-256s the tuple `(account_id, environment_id_or_"auto", cloud_name, PROTOCOL_VERSION, registration_id)`. The `registration_id = Uuid::new_v4()` (`cloud.rs:333`) is generated **once per `run_cloud_executor` invocation** and reused across reconnects — which means the *same* exec-server process re-registers under the same idempotency id, but a fresh process on the same box gets a fresh id. That's the right semantic: cloud side dedupes reconnects but allows operator restarts. Worth a one-line comment at line 333 documenting this so a future refactor doesn't accidentally move it inside the `loop {}` and break dedup.

- **Reconnect loop has no jitter.** `cloud.rs:344-346` does `backoff = (backoff * 2).min(Duration::from_secs(30))` and `sleep(backoff).await` after each disconnect. If the cloud-environments service goes down and 1000 exec-servers all reconnect at the same `2s, 4s, 8s, ..., 30s` cadence, that's a thundering herd. Add a `±25%` jitter — trivial with `rand::thread_rng().gen_range(0.75..=1.25)` against the duration. Not blocking for v1 but worth a follow-up.

- **`recover_unauthorized` re-tries even on hard failures.** I can't see the impl in the visible diff, but the post-`401` semantics need to fail closed if `auth_manager.reload()` produced no auth or non-ChatGPT auth — otherwise the second `post_json` attempt will hit the same `CloudEnvironmentAuth` error from `cloud_environment_chatgpt_auth` (`cloud.rs:355-372`) and return that as the user-facing error, swallowing the original `401` body. Worth confirming the auth-error message preserves the original HTTP status context.

- **`ExecServerError::CloudEnvironmentRequest(#[from] reqwest::Error)`** at `client.rs:266` is good — but `reqwest::Error::Display` can leak the request URL with bearer token in the redacted form. `reqwest` does redact by default (`<REDACTED>`), so this is fine, but worth a doc-comment on the variant noting that the wrapped error will not include the bearer token (so future changes don't accidentally introduce one via `.error_body()` chaining).

- **`reqwest::Client::new()` per `CloudEnvironmentClient`** at `cloud.rs:243` is fine for the singleton case (one client per `run_cloud_executor` invocation). Not a connection-pool waste.

- **CLI override threading is correct.** The change at `cli/src/main.rs:1172` `run_exec_server_command(cmd, &arg0_paths, root_config_overrides.clone())` propagates the `-c` flag overrides into the cloud-mode `Config::load_with_cli_overrides(cli_overrides).await?` at line 1278. Without this, operators would not be able to point cloud-mode at a non-default config (e.g. for staging vs. prod auth profiles).

- **README addition at `exec-server/README.md:24-30`** is accurate: documents the new flag set, mentions the env-var fallback, calls out the ChatGPT-auth requirement. Adequate for the v1 surface.

- **`wiremock` added as dev-dep** (`Cargo.toml:56`) — implies `cloud.rs` has unit tests using a mock HTTP server. Not visible in the truncated diff but the dep+ feature shape (`reqwest` gains the `"json"` feature) implies registration-flow integration tests exist. Reviewer should spot-check coverage on the `401`-then-success path.

## Risk

Medium — this is 400 LOC of new networking + auth surface, but it's gated behind an explicit `--cloud` flag, so the local websocket mode (the entire current user base) is unaffected. The risk is concentrated in the cloud-mode happy path going wrong in ways that block exec-server from starting. The reconnect loop running without jitter is the only operational concern at scale; the rest is hardening for the long tail.

## Nits

1. Doc-comment the per-process `registration_id` lifetime at `cloud.rs:333`.
2. Add ±25% jitter to the reconnect backoff at `cloud.rs:344-346`.
3. Confirm `recover_unauthorized` preserves original `401` status context in the user-facing error path.
4. One-liner doc-comment on `ExecServerError::CloudEnvironmentRequest` noting the bearer-token redaction guarantee.

## Verdict

**merge-after-nits** — solid first cloud-executor cut with the right shape (bounded auth retry, exponential reconnect, idempotency-id derived from auth+config+protocol-version, sha256 hash for collision resistance). The four nits above are operational hardening, not correctness blockers; the maintainer can reasonably land this and follow up.

## What I learned

When a service registers itself with a control plane and then reconnects to a returned rendezvous URL, the *idempotency key lifetime* is the most consequential design decision in the whole flow. Per-process (this PR) means restart-creates-new-executor; per-config-tuple would mean restart-resumes-old-executor; per-hostname would mean re-image-creates-new-executor. The first is the right default for an operator-controlled CLI. The pattern of SHA-256ing `(identity, config, protocol_version, lifetime_token)` to derive the key is generalizable — `PROTOCOL_VERSION` in the hash means a future protocol bump auto-invalidates dedupe state without operator intervention.
