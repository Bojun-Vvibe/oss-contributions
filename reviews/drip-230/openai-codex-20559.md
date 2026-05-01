# openai/codex #20559 — config: add strict config parsing

- **PR**: https://github.com/openai/codex/pull/20559
- **Head SHA**: `a89ac00f54e5218068b56727dcf506a1efd963ea`
- **Files reviewed**: `codex-rs/Cargo.toml`, `codex-rs/Cargo.lock`, `codex-rs/app-server/src/lib.rs`, `codex-rs/chatgpt/src/apply_command.rs`, `codex-rs/cli/src/debug_sandbox.rs`, plus the wide CLI/TUI/exec/MCP/login surfaces
- **Date**: 2026-05-01 (drip-230)

## Context

By design, Codex's TOML config loader silently ignores unknown fields so that
older/newer config files keep parsing across version skew. That same leniency turns typos
(`temperture = 0.7`, `[modle_providers.openai]`) into invisible no-ops. This PR adds an
opt-in `--strict-config` flag that flips the loader to **report** unknown fields with
their source locations while keeping the default permissive behavior unchanged.

The implementation choice is the right one: rather than sprinkling
`#[serde(deny_unknown_fields)]` across every `ConfigToml` substruct (which would also
break upgrade compatibility everywhere by default), the PR pulls in the
`serde_ignored` 0.1.14 crate. `serde_ignored` wraps an existing serde deserializer and
captures the path of every field serde *would* have ignored — so the same parse runs
permissively, and strict mode just reads the captured-path list afterward.

## Diff highlights

- `codex-rs/Cargo.toml:339`: `serde_ignored = "0.1.14"` workspace dep, propagated into
  `codex-core`'s entry in `Cargo.lock:2333-2336`.
- `codex-rs/app-server/src/lib.rs:413-423`:
  ```diff
  -    loader_overrides: LoaderOverrides,
  +    mut loader_overrides: LoaderOverrides,
  +    loader_overrides.strict_config |= cli_config_overrides.strict_config;
  ```
  This is the correct merge semantics: the in-process app-server entrypoint gets `strict_config`
  from either the CLI flag or a programmatic loader-override, with `|=` so neither side
  silently disables the other.
- `codex-rs/chatgpt/src/apply_command.rs:23-37` switches `Config::load_with_cli_overrides`
  to `load_with_cli_overrides_and_loader_overrides` so the apply subcommand actually
  honors the flag.
- `codex-rs/cli/src/debug_sandbox.rs:190-197, 629-720` threads a fresh `strict_config: bool`
  arg through `load_debug_sandbox_config` → `load_debug_sandbox_config_with_codex_home` →
  `build_debug_sandbox_config`, with the conditional at `:709-720`:

  ```rust
  if strict_config || matches!(managed_requirements_mode, ManagedRequirementsMode::Ignore) {
      builder = builder.loader_overrides(LoaderOverrides {
          strict_config,
          ignore_managed_requirements: matches!(...),
          ..Default::default()
      });
  }
  ```

  The previous `if let ManagedRequirementsMode::Ignore = ... { builder = builder.loader_overrides(...) }`
  branch is preserved as the `||` arm — no regression for the
  ignore-managed-requirements path; strict mode just adds a second OR condition that hits
  the same loader_overrides setter.

## Observations

1. **The choice to validate at *every* loader site (system / user / trusted-project /
   legacy-managed / macOS MDM) is what makes this useful.** A typo in `~/.codex/config.toml`
   was the main motivation, but typos in MDM-pushed config files are an even more
   load-bearing reason — those are deployed by an admin who can't reproduce the user's
   environment, so silent ignore there is genuinely dangerous. The PR description names
   all five sources explicitly; the diff confirms the threading.

2. **`serde_ignored` keeps the existing typed-TOML diagnostic ordering.** The PR
   description says "preserving the existing parse and typed TOML diagnostics before
   reporting unknown fields" — that's the right ordering. If the file fails to parse at
   all, the user gets the existing toml error; only on successful parse do unknown-field
   diagnostics fire. So adding strict mode doesn't change error messages for the broken
   case.

3. **`CliConfigOverrides { raw_overrides: ..., ..Default::default() }`** at
   `app-server-test-client/src/lib.rs:2128` — the `..Default::default()` is the *new*
   field-spread that future-proofs the test client against any further `CliConfigOverrides`
   field additions. Good housekeeping.

4. **MODULE.bazel.lock churn is expected.** The bazel lock additions for
   `serde_ignored_0.1.14` and the `windows-sys` version drift in `Cargo.lock:5275, 9292,
   13946` are byproducts of the `cargo update` that came with the new dep — none of them
   are user-facing and they don't change runtime behavior. (Worth noting: the
   `windows-sys 0.61.2` reshuffle is co-incidental to this PR but should be called out in
   the PR body so a reviewer doesn't chase it.)

## Risks / nits

- **No `--strict-config` *positive* unknown-key test in the diff snippet I read** — the
  PR description claims "Added coverage for top-level and nested unknown config fields,
  including source-location reporting" so it likely exists; worth confirming it covers
  both per-source paths (user vs system vs MDM) rather than only one source.
- **`loader_overrides.strict_config |= cli_config_overrides.strict_config`** at
  `app-server/src/lib.rs:423` correctly handles the OR-merge but doesn't surface a
  diagnostic when *both* sides ask for strict mode — fine, since they agree, but if a
  programmatic caller expected to *disable* what the CLI requested, there's no escape
  hatch. A docstring on `LoaderOverrides::strict_config` saying "OR-merged with CLI flag,
  cannot be disabled by loader_overrides if CLI requested it" would prevent confusion.
- **Cross-cutting threading** (CLI / TUI / exec / MCP server / apply / login / marketplace
  / app-server / debug-sandbox all touched) is exactly the sort of change where a reviewer
  wants a single helper that wraps `Config::load_with_cli_overrides_and_loader_overrides`
  + the strict_config plumbing, so the next "thread a load-time flag through everything"
  PR doesn't have to repeat the same 5-call shape. Not blocking.

## Verdict

merge-after-nits
