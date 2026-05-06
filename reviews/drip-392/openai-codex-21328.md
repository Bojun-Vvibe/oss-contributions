# openai/codex #21328 — test: isolate app-server-client in-process test state

- PR: https://github.com/openai/codex/pull/21328
- Head SHA: `35c1133d45d10931914dbb88a1246a195d025ff6`
- Base: `main`
- Size: +57 / -6 across 3 files
- Files:
  - `codex-rs/Cargo.lock` (+1/-0)
  - `codex-rs/app-server-client/Cargo.toml` (+1/-0)
  - `codex-rs/app-server-client/src/lib.rs` (+55/-6)

## Verdict
**merge-as-is**

## Rationale
Test-isolation fix that closes a real cross-test pollution risk. The PR body cites `app-server/src/in_process.rs:368-373` as the source of the issue: when `state_db` is `None`, the in-process app server falls back to `init_state_db_from_config(...)`, which means tests sharing an ambient `codex_home` would also share persisted state across runs. That's a textbook flaky-test source.

The fix is the right shape:

1. **Per-test temporary `codex_home`** via `TempDir::new()` at `codex-rs/app-server-client/src/lib.rs:990-991`. Each test now gets a fully isolated config root.
2. **Explicit `state_db` initialization** at `:991-993` — `init_state_db_from_config(config.as_ref()).await.expect(...)`, then passed in as `state_db: Some(state_db)` at `:1006`. This bypasses the fallback path entirely so tests are no longer subject to the implicit-init contract.
3. **`TestClient` wrapper** at `:1009-1029` holds the `_codex_home: TempDir` alongside the `client`. The leading underscore is the right idiom — `TempDir`'s `Drop` runs cleanup, and storing it in the struct keeps the temp dir alive for the client's full lifetime. Without this wrapper, the `TempDir` would `Drop` at the end of `start_test_client_with_capacity` and the test would race against directory cleanup.
4. **`Deref` impl** at `:1014-1020` means existing test bodies that called methods on `InProcessAppServerClient` continue to work unchanged on `TestClient` — minimal blast radius across the test file.

The `build_test_config_for_codex_home` helper at `:983-997` is defensive in a sensible way: tries `ConfigBuilder::default().codex_home(...).build()` first, falls back to `Config::load_default_with_cli_overrides_for_codex_home` on error. That keeps the test harness resilient if the `ConfigBuilder` API gains required fields later.

`tempfile` added as a dev-dependency at `Cargo.toml:36` — already in workspace `Cargo.lock`, no new transitive deps.

## Specific lines
- `codex-rs/app-server-client/src/lib.rs:983-997` — `build_test_config_for_codex_home` helper. Defensive fallback is correct.
- `codex-rs/app-server-client/src/lib.rs:990-1008` — per-test temp home + state-db init. Correct.
- `codex-rs/app-server-client/src/lib.rs:1009-1029` — `TestClient` wrapper with `_codex_home: TempDir`, `Deref`, and `shutdown`. Correct lifetime management.
- `codex-rs/app-server-client/Cargo.toml:36` — `tempfile = { workspace = true }` dev-dep. Correct scope.

## Nits (none worth blocking)
- The fallback branch in `build_test_config_for_codex_home` could log/trace which path was taken, but for a test harness that's overkill.
- The `expect("temp dir")` and `expect("state db should initialize for in-process test")` strings are fine — they only fire when the test environment itself is broken.

Ship it.
