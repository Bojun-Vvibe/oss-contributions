# openai/codex#21323 — [codex] support executor registry environments

- URL: https://github.com/openai/codex/pull/21323
- PR: #21323
- Author: miz-openai (Michael Zeng)
- Head SHA: `c07b2cb9c76859c8a2af9c82c2a3d36e408e9f7e`
- State: OPEN  | +102 / -78

## Summary

Adds dynamic remote-environment registration to `EnvironmentManager`: previously `environments: HashMap<String, Arc<Environment>>` was constructed once and frozen. This PR wraps it in `RwLock` and exposes `upsert_remote_environment(id, exec_server_url)` so the executor-registry path can register named remote environments at runtime without restarting the manager. Also subtractive cleanup in `remote.rs` — drops the unused `BTreeMap`/`Serialize`/`Value`/`sha2`/`Uuid`/`PROTOCOL_VERSION` machinery and inlines the now-only caller of the deleted `post_json` helper.

## Specific references

- `exec-server/src/environment.rs:3`: `use std::sync::RwLock`.
- `environment.rs:39`: `environments: HashMap<...>` → `environments: RwLock<HashMap<...>>`. Constructor sites at `:62-67` and `:163` updated to wrap.
- `environment.rs:185-191`: `get_environment` now takes a read lock, with explicit `unwrap_or_else(std::sync::PoisonError::into_inner)` to swallow poisoning rather than panic.
- `environment.rs:193-225`: new `upsert_remote_environment` — validates non-empty id, normalizes URL via `normalize_exec_server_url`, rejects "disabled" sentinel, builds an `Environment::remote_inner(...)` reusing the local-runtime paths, then write-locks and `insert`s.
- `environment.rs:570-606`: two new `#[tokio::test]`s covering insert+update (asserts `Arc::ptr_eq` is *false* across versions, so consumers don't accidentally hold a stale Arc) and empty-URL rejection.
- `exec-server/src/remote.rs:1-19`: drops 6 `use` statements (`BTreeMap`, `Serialize`, `Value`, `sha2::Digest`, `Uuid`) plus `PROTOCOL_VERSION` constant — net subtractive.
- `remote.rs:46-67`: `register_executor` now takes `&str executor_id` directly instead of `&ExecutorRegistryRegisterExecutorRequest`, inlines the URL formatting, and the generic `post_json<T,R>` helper is gone (the only caller is now hand-rolled).

## Concerns / nits

- **`unwrap_or_else(PoisonError::into_inner)` on every read** is a deliberate "never panic on poisoned lock" stance — fine for a long-running server but worth a one-line module comment or a `tracing::warn!` on the recovery path so a real poisoning incident isn't invisible.
- `upsert_remote_environment` does not also let the caller atomically *set* the new environment as default. Today `default_environment` is `Option<String>` set once at construction. If executor registry needs to flip the default after a runtime register, that's a follow-up — call out in the PR body whether default-flipping is intentionally out of scope.
- The test at `:572-595` asserts `!Arc::ptr_eq(&first, &second)` on update but doesn't assert that any *outstanding* `Arc<Environment>` from the first lookup remains usable (it should, since `Arc` clones survive the map removal). Add a one-line assertion to lock the no-invalidation contract.
- `remote.rs` cleanup is unrelated to the executor-registry feature — split commit would help reviewers, but it's small enough to land together.
- No test for the "executor registry path actually calls upsert" — that's presumably exercised in a higher-level test elsewhere; worth a comment pointer.

## Verdict

**merge-after-nits (man)** — clean refactor with thoughtful tests; missing one assertion and a comment on poisoning policy.
