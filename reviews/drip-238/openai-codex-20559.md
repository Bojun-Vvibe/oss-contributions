# openai/codex#20559 — config: add strict config parsing

- **PR**: https://github.com/openai/codex/pull/20559
- **Author**: bolinfest
- **Head SHA**: `a89ac00f54e5218068b56727dcf506a1efd963ea`
- **Files (top)**: `codex-rs/Cargo.toml`, `codex-rs/Cargo.lock`, `codex-rs/MODULE.bazel.lock`, `codex-rs/app-server/src/lib.rs`, `codex-rs/chatgpt/src/apply_command.rs`, `codex-rs/cli/src/debug_sandbox.rs` (+ test fixtures)
- **Verdict**: **merge-after-nits**

## Context

Today, unknown keys in `~/.codex/config.toml` (typos like `mode_provider` instead of `model_provider`, deprecated keys, fat-fingered nested tables) are silently ignored by `serde`'s default `#[serde(deny_unknown_fields = false)]` behavior. Users debugging "why isn't my override applying?" have no signal that the key didn't bind. The PR introduces an opt-in `strict_config` flag (CLI `--strict-config` / `LoaderOverrides::strict_config`) that wires `serde_ignored` into the TOML deserializer to surface unknown keys as a load-time error.

## What's right

- **`serde_ignored` is the right primitive.** Pulled in at `Cargo.toml:339` with workspace pin `serde_ignored = "0.1.14"` and the corresponding `Cargo.lock`/`MODULE.bazel.lock` entries land cleanly with no version-tree disruption beyond the new crate. `serde_ignored::deserialize` wraps any deserializer and invokes a callback for each path that wasn't consumed by the target struct — the canonical pattern for this exact problem and avoids the `#[serde(deny_unknown_fields)]` propagation tax across every nested config struct.
- **OR-merge of CLI + loader override at one place.** `app-server/src/lib.rs:423` adds `loader_overrides.strict_config |= cli_config_overrides.strict_config;` — meaning either surface (top-level CLI flag or programmatic loader override from a host) can opt in, and neither can opt the other out. Correct precedence direction for a strictness flag (more-strict wins).
- **Threading through the debug-sandbox path is correct.** `debug_sandbox.rs:629-712` adds a `strict_config: bool` parameter to `load_debug_sandbox_config`, `load_debug_sandbox_config_with_codex_home`, and `build_debug_sandbox_config`, plus updates the builder branch at `:706-716` to OR `strict_config || ManagedRequirementsMode::Ignore` so the prior `Ignore`-only branch isn't accidentally inverted. The five test-arm updates at `:787-1018` symmetrically pass `/*strict_config*/ false` so existing test semantics are preserved (the new flag defaults off).
- **`CliConfigOverrides::raw_overrides` initialization at `app-server-test-client/src/lib.rs:2127` adds `..Default::default()`** which gracefully forward-compats the test client against future fields on `CliConfigOverrides` (including the new `strict_config: bool`) — the right discipline for test scaffolding.
- **`apply_command.rs:26-35` flips from `Config::load_with_cli_overrides` to `Config::load_with_cli_overrides_and_loader_overrides`** with an inline `LoaderOverrides { strict_config: apply_cli.config_overrides.strict_config, ..Default::default() }` — the `..Default::default()` on the struct literal again forward-compats against future loader-override fields.

## Risks / nits

- **The `windows-sys` version churn at `Cargo.lock:5275`/`:9292`/`:13946` (0.61.2 ↔ 0.52.0 / 0.45.0 ↔ 0.61.2 / 0.48.0 ↔ 0.61.2) is unrelated to strict-config and is dependency-graph noise from `cargo update` running alongside the `serde_ignored` add.** Same with the `base64 0.21.7 → 0.22.1` shift in `keyring`. Reviewers will burn time tracing whether these are intentional. Recommend either (a) split `cargo update` side-effects into a separate prep PR and rebase this one onto a clean lockfile, OR (b) explicitly call out in the PR body that these shifts are incidental.
- **No test pinning that strict mode actually rejects an unknown key.** The diff adds the plumbing but I don't see a `#[test] fn strict_mode_rejects_unknown_top_level_key()` or similar. Without a test the regression surface is "did anyone forget to pass `strict_config` through one of the four entry points?" — easy to drift. Recommend at minimum one positive test (unknown key triggers error in strict mode) and one negative test (same key is silently ignored in non-strict mode).
- **The strict-mode error message shape isn't in the diff.** What does the user actually see when they typo a key? `serde_ignored`'s callback gives a path string; the callback handler should produce a message like `"Unknown config key 'mode_provider' at config.toml:42 (did you mean 'model_provider'?)"`. Without confirming the rendered error is actionable, strict mode is a regression for users who flip it on and get a `serde` debug-format error dump.
- **`debug_sandbox.rs:706-716` builder branch logic** combines two booleans: `if strict_config || matches!(managed_requirements_mode, ManagedRequirementsMode::Ignore)`. The old code was `if let ManagedRequirementsMode::Ignore = managed_requirements_mode { ... loader_overrides ... }`. Subtle behavior shift: previously, `Ignore` mode set `ignore_managed_requirements: true` and `..Default::default()` (so `strict_config` was implicitly false). Now, the same `Ignore` mode sets `ignore_managed_requirements: matches!(...)` AND `strict_config: strict_config`. Functionally equivalent when `strict_config = false` but the structural change is worth a unit test to pin that "strict_config + Ignore" combines correctly (both effects active).

## Verdict

**merge-after-nits.** Right primitive, right plumbing, right precedence. Wants (a) the lockfile churn split or explained, (b) at least one positive + one negative test, and (c) confirmation the rendered strict-mode error is user-actionable. Once those land this is a safe, opt-in strictness improvement.
