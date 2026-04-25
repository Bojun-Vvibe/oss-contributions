---
pr: 19575
repo: openai/codex
sha: 1869c17b836a65a22eed1e66fe389f572e525959
verdict: merge-after-nits
date: 2026-04-26
---

# openai/codex#19575 — Add cloud executor registration to exec-server

- **URL**: https://github.com/openai/codex/pull/19575
- **Author**: miz-openai

## Summary

Adds a `--cloud` mode to `codex exec-server` that, instead of
listening on a local websocket, registers itself with a cloud
"environments" service via `POST /api/cloud/executor`, then dials
the rendezvous websocket URL the service returns and serves the
same `ConnectionProcessor` over it. Reconnects with exponential
backoff (1s → 30s) on disconnect.

New crate dependencies: `codex-login` (for `AuthManager`/
`CodexAuth`), `sha2` (idempotency-id derivation), and `wiremock`
in dev-deps.

## Reviewable points

- `codex-rs/cli/src/main.rs:454-467` — four new `clap` flags
  (`--cloud`, `--cloud-base-url`, `--cloud-environment-id`,
  `--cloud-name`) plumbed through `ExecServerCommand`. Standard
  pattern; the `--cloud-base-url` precedence — flag → env var
  `CODEX_CLOUD_ENVIRONMENTS_BASE_URL` → error — is reasonable.

- `codex-rs/cli/src/main.rs:1257-1281` — when `cmd.cloud` is set,
  the function builds a fresh `Config` from CLI overrides and a
  `Arc<AuthManager>` with `enable_codex_api_key_env=false`. The
  `false` is important: cloud mode requires ChatGPT auth (see
  `cloud.rs:343`), so allowing API-key auth here would let a user
  start the executor and then fail at first request. Forcing it
  off at construction time is the right call.

- `codex-rs/exec-server/src/cloud.rs:240-265` — `run_cloud_executor`
  loops forever: register → connect_async → run_connection → sleep
  backoff. There is no cap on the loop, no SIGINT/SIGTERM
  awareness, no graceful drain. For a long-running daemon this is
  probably fine (systemd will signal kill), but a `tokio::select!`
  on a `ctrl_c()` future would let in-flight requests finish
  cleanly. Not blocking.

- `codex-rs/exec-server/src/cloud.rs:160-181` —
  `default_idempotency_id` hashes
  `account_id || env_id || cloud_name || PROTOCOL_VERSION`. Good:
  re-registering with the same args reuses the executor record.
  But: if the user changes `--cloud-name` between restarts, the
  old executor record will linger server-side. The PR description
  should mention server-side TTL or a manual deregister path; if
  neither exists, repeated `--cloud-name` changes will accumulate
  ghost executors.

- `codex-rs/exec-server/src/cloud.rs:283-313` —
  `cloud_environment_chatgpt_auth` does an interesting "reload
  once if auth missing" loop. Correct, but the `reloaded = true`
  flag means we get exactly one retry. If `auth.get_account_id()`
  returns `None` after the reload too, we proceed to
  `chatgpt_account_id(&auth)?` which then returns
  `CloudEnvironmentAuth("waiting for a ChatGPT account id")`.
  That error message says "waiting" but the code does *not* wait
  or retry — it bubbles up and the outer loop will hot-spin on
  re-registration with backoff. Either rename the message
  ("missing ChatGPT account id") or actually wait.

- `codex-rs/exec-server/src/cloud.rs:213-216` — the `unreachable!`
  after the `for attempt in 0..=1` loop is correct (loop body
  always returns or continues, and continue only fires when
  `attempt == 0`). Slightly fragile to future edits; a `Result`
  return with a fallthrough error would be safer but the current
  pattern is idiomatic Rust.

- Backoff at `cloud.rs:262` does `(backoff * 2).min(30s)` after
  *both* successful runs and failures. After a normal disconnect
  + immediate reconnect, the backoff stays at the last doubled
  value rather than resetting on the next successful connect.
  The reset to 1s on `Ok(...)` at line ~256 is correct, but it's
  doing reset-then-double-after-disconnect, which lands the next
  reconnect attempt at 2s. Acceptable; a comment would help.

## Rationale

Solid feature implementation with proper auth-mode enforcement,
bounded retries, and idempotent registration. The cloud-name
ghost-executor question and the misleading "waiting" error
message are nits worth fixing. The lack of a graceful shutdown
path is the only structural concern, and it's not blocking for
v1.

## What I learned

`AuthManager::shared_from_config(&config, false)` is the
chokepoint that prevents API-key auth from sneaking past
ChatGPT-only features. Threading that boolean from CLI command
type to the AuthManager constructor is more robust than checking
`auth.is_chatgpt_auth()` at every call site, which is what the
file currently does anyway as a defense-in-depth.
