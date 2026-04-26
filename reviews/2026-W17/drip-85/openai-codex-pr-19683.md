# openai/codex PR #19683 — test: harden app-server integration tests

- **Repo:** openai/codex
- **PR:** [#19683](https://github.com/openai/codex/pull/19683)
- **Files:** 14 files modified across `codex-rs/app-server/`, `codex-rs/core/tests/common/`, and `codex-rs/rmcp-client/`

## Context

Integration tests against the real `codex-app-server` binary have been
flaky/slow because every test process kicks off plugin startup tasks
(model warmups, plugin manager bootstrapping) on connect, even when
the test only exercises the JSON-RPC surface and never touches a
plugin. This PR introduces a debug-only CLI flag that disables those
startup tasks and threads it through every test-spawn helper.

## What the diff actually does

**New runtime-options enum at `app-server/src/lib.rs:362-381`:**
```rust
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum PluginStartupTasks {
    Start,
    Skip,
}

#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub struct AppServerRuntimeOptions {
    pub plugin_startup_tasks: PluginStartupTasks,
}

impl Default for AppServerRuntimeOptions {
    fn default() -> Self {
        Self { plugin_startup_tasks: PluginStartupTasks::Start }
    }
}
```

**New entry-point `run_main_with_transport_options`** at `lib.rs:411-434`
that takes `runtime_options: AppServerRuntimeOptions`. The original
`run_main_with_transport` is preserved as a thin shim that delegates
with `AppServerRuntimeOptions::default()` (`:370-409`), so all
existing callers compile unchanged.

**Hidden CLI flag at `main.rs:38-50`:**
```rust
/// Hidden debug-only test hook used by integration tests that spawn the
/// production app-server binary.
#[cfg(debug_assertions)]
#[arg(long = "disable-plugin-startup-tasks-for-tests", hide = true)]
disable_plugin_startup_tasks_for_tests: bool,
```
Wired into `runtime_options.plugin_startup_tasks` only when
`debug_assertions` (i.e., never in release builds) at `main.rs:62-65`.

**Threading through `MessageProcessorArgs`** (`message_processor.rs:259-281`)
adds a `plugin_startup_tasks: crate::PluginStartupTasks` field, then
gates the warmup call at `message_processor.rs:316-322`:
```rust
if matches!(plugin_startup_tasks, crate::PluginStartupTasks::Start) {
    thread_manager
        .plugins_manager()
        .maybe_start_plugin_startup_tasks_for_config(&config, auth_manager.clone());
}
```

**Test-side wiring**:
- `tests/common/mcp_process.rs:107` exports the literal CLI arg:
  `pub const DISABLE_PLUGIN_STARTUP_TASKS_ARG: &str = "--disable-plugin-startup-tasks-for-tests";`
- `McpProcess::new` now defaults to passing the flag (`:115`), with
  a new opt-in `new_with_plugin_startup_tasks` for tests that
  explicitly need warmup behavior.
- `RUST_LOG` lowered from `"info"`/`"debug"` to `"warn"` in three
  test spawn helpers (`mcp_process.rs:159`,
  `connection_handling_websocket.rs:399,531`).
- Two websocket spawn helpers
  (`connection_handling_websocket.rs:392-401` and `:526-535`) and
  every other test that spawns the binary directly are updated to
  pass `DISABLE_PLUGIN_STARTUP_TASKS_ARG` explicitly.

**`in_process.rs:418`** also gets `plugin_startup_tasks: crate::PluginStartupTasks::Start`
in the in-process path so the in-process binding is unaffected
(real CLI/SDK users still get warmups).

## Strengths

- The opt-in hidden flag pattern is the right shape: zero impact
  on release builds (`#[cfg(debug_assertions)]`), invisible in
  `--help` (`hide = true`), and the const-string export at
  `mcp_process.rs:107` means there's exactly one place to change
  the literal flag name.
- Backwards compatibility preserved via the
  `run_main_with_transport` → `run_main_with_transport_options`
  shim. No external caller needs to update.
- The `PluginStartupTasks` enum at `:363-366` is two-state and
  `Copy`, so threading it through `MessageProcessorArgs` doesn't
  cost meaningful complexity — just one field.
- Lowering `RUST_LOG` from `info`/`debug` to `warn` in test helpers
  cuts CI log volume, which is a real flake reducer when tests
  match on specific log lines.
- The `tracing_tests.rs:288-292` build_test_processor helper is
  updated to pass `PluginStartupTasks::Start` so its semantics are
  preserved (it's a unit-test-of-tracing harness, not an
  integration test, so warmup-skip would change behavior).

## Risks / nits

1. **`plugin_startup_tasks` field on `MessageProcessorArgs` defaults
   to nothing** — every call site has to explicitly pass
   `PluginStartupTasks::Start`. The diff updates two such sites
   (`in_process.rs:418`, `tracing_tests.rs:291`); a future caller
   that forgets won't get a compile error (it's a struct field,
   not a builder default). Consider a `#[builder(default = …)]`
   pattern or implementing `Default` on `MessageProcessorArgs`.
2. **Single boolean instead of separate per-warmup flags** — if a
   future test wants to skip *only* model warmup but keep plugin
   manager bootstrap, this two-state enum can't express that. The
   enum shape leaves room for `PluginStartupTasks::SkipWarmupKeepBootstrap`
   without breaking callers; mention that in a doc-comment so future
   contributors don't reach for a `bool` and miss the precedent.
3. **`new_with_plugin_startup_tasks` opt-in** at `mcp_process.rs:120`
   has no callers in this PR's diff. Either add a test that
   exercises the warmup path on purpose, or document that the
   opt-in exists for tests we know we'll want to add later.
   Otherwise it becomes a one-off helper that drifts.
4. **`new_with_args` rebuild at `:127-130`**:
   ```rust
   let mut all_args = vec![DISABLE_PLUGIN_STARTUP_TASKS_ARG];
   all_args.extend_from_slice(args);
   ```
   This silently injects the flag into every `new_with_args` call.
   If a future test wants the warmup behavior *and* wants to pass
   custom args, they'll have to use `new_with_plugin_startup_tasks`
   + raw spawn rather than `new_with_args`. Add a doc-comment to
   `new_with_args` calling out the implicit flag injection.
5. **`RUST_LOG` lowered to `warn` in `connection_handling_websocket.rs`**
   at lines 401 and 531 (was `debug`) — that's a 5-level drop. If
   there's any test currently asserting on `info`-level log lines,
   it'll silently start passing-without-asserting because the line
   never fires. Worth a `git grep` over `tests/` for log-string
   assertions before merge.
6. **No regression test** that asserts the flag actually skips
   warmup. `assert!(plugin_manager.warmup_count() == 0)` after
   spawning with the flag would lock the contract; without it,
   a future refactor that re-introduces unconditional warmup
   would be silently undetected by the very tests this PR is
   supposed to harden.
7. **`pub use mcp_process::DISABLE_PLUGIN_STARTUP_TASKS_ARG`** at
   `tests/common/lib.rs:28` re-exports the const. Good. But
   `connection_handling_websocket.rs:4` imports it from
   `app_test_support`, which is the `tests/common/lib.rs` crate
   alias — verify the alias is set in `tests/Cargo.toml` (not
   visible in the diff).

## Verdict

`merge-after-nits`

Solid, well-scoped test-infra hardening. The
`PluginStartupTasks::{Start,Skip}` two-state enum threading and
`#[cfg(debug_assertions)]` flag gate are the right design — no
release-build risk, no public API churn. Three nits worth fixing
before merge: add a regression test that the flag actually skips
warmup, document the implicit flag injection in `new_with_args`,
and grep `tests/` for log-string assertions that the
`debug→warn` drop might silently bypass. The unused
`new_with_plugin_startup_tasks` opt-in helper can stay if there's
a follow-up issue tracking its first user.
