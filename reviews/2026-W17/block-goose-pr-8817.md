---
pr: 8817
repo: block/goose
sha: b9b0e362a3e09d971f23072c2c064c0381990593
verdict: merge-after-nits
date: 2026-04-26
---

# block/goose #8817 — refactor(logging): consolidate logging setup into shared helper in goose crate

- **Author**: r0x0d
- **Head SHA**: b9b0e362a3e09d971f23072c2c064c0381990593
- **Link**: https://github.com/block/goose/pull/8817
- **Size**: ~415 diff lines across `Cargo.lock`, `crates/goose-cli/src/logging.rs`, `crates/goose-server/src/logging.rs`, `crates/goose/Cargo.toml`, `crates/goose/src/logging.rs`.

## Scope

Pulls the duplicated tracing-subscriber wiring out of `goose-cli/src/logging.rs` and `goose-server/src/logging.rs` into a single `goose::logging::build_logging_subscriber(&LoggingConfig)` helper. The two callers shrink to ~15 lines each; the helper handles file appender, JSON formatting, env-filter merging, optional console layer (stderr, pretty), `#[cfg(feature = "otel")]` OTLP layers, and Langfuse layer.

## Specific findings

- `crates/goose/src/logging.rs:6-92` — new `LoggingConfig<'a>` struct (component, name, extra_directives, console: bool) and `build_logging_subscriber` returning `impl SubscriberInitExt + Send + Sync + 'static`. Returns the subscriber unstarted so callers can choose `try_init()` (server) or `set_default()` (cli's force-test path). Good API choice — keeps the `Once`-guard at the caller where it belongs.
- `crates/goose/src/logging.rs:21-35` — `build_env_filter` reads `RUST_LOG` first, falls back to `mcp_client=info, goose=info, WARN`, then layers in caller-supplied `extra_directives`. The CLI passes `["goose_cli=info"]`, the server passes `["goose_server=info", "tower_http=info"]`. Behavior matches the previous per-crate filters at `goose-cli/src/logging.rs:14-21` (pre-diff) and `goose-server/src/logging.rs:35-44` (pre-diff). Good.
- `crates/goose/src/logging.rs:78-85` — OTLP and Langfuse layers are appended unconditionally based on `#[cfg(feature = "otel")]` and `Some(langfuse) = ...create_langfuse_observer()`. Note the change in semantics for the **CLI**: previously `otlp::init_otlp_layers` was gated behind `if !force` at `goose-cli/src/logging.rs:46`, so the test-mode `setup_logging_internal(name, true)` path skipped OTLP. The shared helper has no such gate, so test runs that happen to have OTLP env vars set will now spin up an OTLP exporter. This is probably benign (tests set up their own tempdir/env) but it's a subtle behavior shift. Worth either preserving the gate via a `force` field on `LoggingConfig` or asserting the test isolation is sufficient.
- `crates/goose-cli/src/logging.rs:1-50` — caller now ~50 lines (down from ~99), delegates to the helper. The `force` test path keeps its `subscriber.set_default()` + warn/info smoke test. Clean.
- `crates/goose-cli/src/logging.rs:127-138` — the `test_default_filter_avoids_debug_by_default` test (which previously asserted the literal filter string contained `goose=info` and not `goose=debug`) is now reduced to a smoke test (`assert!(...is_ok())`). **This is a real regression in test value** — the original test pinned a contract about default verbosity; the new test only checks that the call doesn't panic. The contract still exists (in `build_env_filter` defaults at `crates/goose/src/logging.rs:24-28`) but is no longer asserted anywhere. Recommend adding an assertion against `build_env_filter(&[]).to_string()` in the goose crate's own test module.
- `crates/goose-server/src/logging.rs` — collapses from ~70 lines to ~15. Loses no functionality. Good.
- `crates/goose/Cargo.toml:103` — adds `env-filter, fmt, json, time` features to `tracing-subscriber` (previously these were transitively pulled in via the CLI/server own tracing-subscriber dep). Correct, since the goose crate now needs them directly.
- `crates/goose/Cargo.toml:196` — adds `tracing-appender.workspace = true`. Correct.
- `Cargo.lock` shows the goose crate now depends on `tracing-appender` directly. Expected.

## Risk

Low-medium. The refactor is mechanical and the call sites match the previous behavior closely. The two real concerns are:
1. The CLI's previous `if !force` gate on OTLP layer initialization is silently dropped — could activate OTLP exporters during `cargo test` on dev machines with OTLP env vars. Easy fix.
2. The default-filter-defaults assertion is lost in the move from `goose-cli` to the smoke test — replace it with a real assertion in the goose crate.

No behavior change for end users. Build-time dep additions are routine.

## Verdict

**merge-after-nits** — restore the OTLP `force` gate (or document it as intentional), and re-add the literal default-filter assertion in the goose crate's logging tests so the "DEBUG-by-default off" contract stays pinned. Otherwise a tidy refactor.
