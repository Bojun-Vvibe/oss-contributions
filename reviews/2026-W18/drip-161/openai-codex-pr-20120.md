# openai/codex PR #20120 — feat(cli): remove sandbox full-auto shortcut

- Repo: `openai/codex`
- PR: https://github.com/openai/codex/pull/20120
- Head SHA: `57f5d14f45eefe42d4225713704db1382e4c7d19`
- State: OPEN, +24/-80 across 4 files

## What it does

Removes the `--full-auto` shortcut from the three `codex debug sandbox <macos|linux|windows>` subcommands now that the equivalent behavior is reachable two cleaner ways: explicit `--permissions-profile` (lands in #20117) and replay-JSON (#20119). Drops the ~80-line `--full-auto`-aware codepath from `codex-rs/cli/src/debug_sandbox.rs` (the option field on `SeatbeltCommand`/`LandlockCommand`/`WindowsCommand`, the `full_auto` member of `DebugSandboxConfigOptions`, and the `create_sandbox_mode(full_auto: bool)` helper that mapped `true → SandboxMode::WorkspaceWrite`, `false → SandboxMode::ReadOnly`).

## Specific reads

- `cli/src/debug_sandbox.rs:51-65` (and the parallel `:97-111`, `:130-142` blocks) — the destructure now omits `full_auto`. This is mechanical, applied identically across all three sandbox-OS handlers, which is the right shape: the field deletion in `DebugSandboxConfigOptions` (line 164-170) forces the compiler to flag any consumer that still reads it, so the parallel changes can't drift.
- `cli/src/debug_sandbox.rs:599-607` — entire `pub fn create_sandbox_mode(full_auto: bool) -> SandboxMode` deleted. The `pub` visibility means downstream consumers (out-of-tree forks, internal tooling) get a hard break at the next bump. Worth a `CHANGELOG`/release-note entry calling out the deletion explicitly, not just "removed --full-auto".
- `cli/src/debug_sandbox.rs:817-828` — the `if config_uses_permission_profiles(&config) { if full_auto { anyhow::bail!("...is only supported for legacy `sandbox_mode` configs...") } }` guard goes away. Pre-PR this was the only place that emitted "use a writable `[permissions]` profile instead" — users on permission-profiles configs who pass `--full-auto` will now hit clap's generic `unrecognized argument` error, which is *less* helpful than the bail message. A short clap `after_help` line on the three subcommands ("note: `--full-auto` was removed; pass `--permissions-profile <name>` instead") would preserve the old guidance.
- The `build_debug_sandbox_config` call (around line 829) now passes `sandbox_mode: None` unconditionally instead of `Some(create_sandbox_mode(full_auto))`. That's correct given the helper's collapse, but means the legacy `sandbox_mode` config path now always sees `None` from this caller — confirm `build_debug_sandbox_config` retains a sensible default for users still on legacy `sandbox_mode = "workspace-write"` configs.

## Risk

- Hard CLI break: any operator script invoking `codex debug sandbox <os> --full-auto …` breaks at the next release. Title says "remove" so that's intentional, but the PR ships without an `--full-auto` → translation hint at the clap layer.
- `create_sandbox_mode` removal also removes a tiny but useful mapping primitive. Nothing else in-tree consumes it (otherwise the build wouldn't pass), but a fork that imported it for parity would silently lose it.

## Verdict

`merge-after-nits` — surface-area cleanup is clean and follows the additive-replacement-then-remove sequencing the PR body claims. Two nits: (1) preserve the user-visible "use `--permissions-profile` instead" guidance via clap `after_help` so the silent `unrecognized argument` outcome doesn't punish users mid-debug, (2) call out the `pub fn create_sandbox_mode` deletion in release notes since that's a downstream-visible API removal not signposted by the PR title.
