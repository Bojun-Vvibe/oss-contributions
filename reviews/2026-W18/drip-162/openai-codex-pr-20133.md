# openai/codex PR #20133 — chore(cli) deprecate --full-auto

- Repo: `openai/codex`
- PR: https://github.com/openai/codex/pull/20133
- Head SHA: `28ef40cfff`
- State: OPEN, +171/-95 across 11 files

## What it does

Walks back the hard `--full-auto` removal that #20120 staged. Rather than deleting the option, this PR:

1. **Fully removes** `--full-auto` from `codex tui` and from `codex sandbox <macos|linux|windows>` (since the equivalent for `tui` is the trust flow, and for `sandbox` is `-c 'sandbox_mode="workspace-write"'`).
2. **Marks `--full-auto` as deprecated** in `codex exec` with a runtime warning, since users may be actively scripting against it. Removal deferred to a future minor.
3. Updates `codex-cli/scripts/run_in_container.sh` to drop `--full-auto` and pass the explicit `--sandbox workspace-write --ask-for-approval on-request` pair.
4. README sections for `codex sandbox <os>` lose the `--full-auto` callout and gain a "to try a writable legacy sandbox mode, pass `-c 'sandbox_mode="workspace-write"'`" note.
5. Replaces the old `--full-auto` codepath in `cli/src/debug_sandbox.rs` with a `cli_overrides_use_legacy_sandbox_mode` helper that detects `-c sandbox_mode=...` overrides and short-circuits the default-read-only fallback.

## Specific reads

- `codex-cli/scripts/run_in_container.sh:95` — direct substitution from `codex --full-auto` to `codex --sandbox workspace-write --ask-for-approval on-request`. Mechanical and correct, but worth noting the old `--full-auto` resolved to a slightly different ask-policy in some configs (`on-failure` vs `on-request`). The script is the canonical way users reach codex inside the container, so a one-line CHANGELOG callout for "container script now uses on-request approvals" would help.
- `codex-rs/cli/src/debug_sandbox.rs:43-115` — destructure of `SeatbeltCommand`/`LandlockCommand`/`WindowsCommand` drops `full_auto` field. The `full_auto: bool` field was deleted from each of the three structs (consistent across the trio). The function signature `run_command_under_sandbox(...)` loses its `full_auto` parameter. Compiler-enforced consistency — if any caller missed it, the build fails. Good.
- `codex-rs/cli/src/debug_sandbox.rs:399-407` — the `pub fn create_sandbox_mode(full_auto: bool) -> SandboxMode` helper that mapped `true → WorkspaceWrite`, `false → ReadOnly` is **deleted**. As called out in the drip-161 review of #20120, this function was `pub` and any out-of-tree fork that imported it for parity now hard-breaks. A release-note bullet ("removed `pub fn create_sandbox_mode`") would close that loop. The PR title is "deprecate" but this particular delete is hard removal.
- `codex-rs/cli/src/debug_sandbox.rs:579-619` — `load_debug_sandbox_config_with_codex_home` now derives `uses_legacy_sandbox_mode_override` via the new helper `cli_overrides_use_legacy_sandbox_mode` (which scans for any `-c` override with key `sandbox_mode`). The two-arm condition `if config_uses_permission_profiles(&config) || uses_legacy_sandbox_mode_override` is clean: "if permissions profiles are configured *or* the user passed an explicit override, trust the resolved config; otherwise force read-only as before." This preserves the historical "default to read-only when invoked as `codex sandbox <os>`" contract that the bare `--full-auto` deletion in #20120 broke.
- `codex-rs/cli/src/debug_sandbox.rs:639-642` — `cli_overrides_use_legacy_sandbox_mode` does a substring scan of override keys. It correctly matches the bare `sandbox_mode` key but **does not match nested overrides like `-c 'sandbox.mode = "..."'`** if such a TOML path becomes valid. Today it's flat, but worth a comment pinning that assumption.
- `codex-rs/cli/src/debug_sandbox.rs:701` — `pretty_assertions::assert_eq` import added but only used in the existing test. Fine.
- `codex-rs/cli/src/debug_sandbox.rs:705-720` — the existing test `legacy_config_falls_back_to_read_only_when_no_override` is updated to drop the `full_auto` arg. There's no *new* test for the new branch (`uses_legacy_sandbox_mode_override = true` should *not* fall back to read-only). That's the load-bearing new logic and it's untested.
- `codex-rs/exec/src/cli.rs` and `:cli_tests.rs` (referenced in file list) — `--full-auto` retained on `codex exec` with a deprecation warning. Couldn't read the diff slice, but the runtime-warn path is the right shape: log once per process at the point we see the deprecated flag, with a pointer to the migration command.
- `codex-rs/tui/src/lib.rs` — `--full-auto` removed entirely. The TUI's existing trust-prompt flow covers the same use-case (running in an untrusted repo), so no migration message is strictly needed but the empty space where the flag used to live is a confused-user opportunity. A clap `after_help` would help.
- `codex-rs/utils/cli/src/shared_options.rs` (referenced) — likely the `SharedOptions` struct lost the `full_auto` field. If any other binary (codex-mcp-server?) consumed `SharedOptions::full_auto`, it'll have been compiler-flagged.

## Risk

1. **`pub fn create_sandbox_mode` deletion** is unannounced in the title (which says "deprecate"). Forks and out-of-tree consumers break. Bullet in release notes.
2. **Asymmetric deprecation** is the right call — keeping `exec` deprecation-warned for one cycle, removing from `tui`/`sandbox` immediately — but the rationale is buried. PR description has it; CHANGELOG should mirror.
3. **No test for the new `uses_legacy_sandbox_mode_override = true` branch**. The whole point of this PR's core logic change is that branch, and it's untested. A 15-line test (`load_debug_sandbox_config_with_codex_home(vec![("sandbox_mode".into(), TomlValue::String("workspace-write".into()))], ...)` → assert resulting `Config.sandbox_mode == WorkspaceWrite`) closes this.
4. **`run_in_container.sh` semantic shift** from `--full-auto` to `--sandbox workspace-write --ask-for-approval on-request`. If `--full-auto` historically resolved to `on-failure` (verify in pre-PR `cli/src/lib.rs`), this changes container behavior. Worth a code archaeology check.
5. The `run_in_container.sh` also now embeds a literal `--ask-for-approval on-request` which is two tokens — make sure they're correctly quoted through the `printf '%q'` loop above (they are, since each `arg` in `$@` is quoted individually, but the new tokens are appended *outside* that loop directly into the `bash -c` string, which is how the original `--full-auto` was inserted too — consistent).

## Verdict

`merge-after-nits` — the design (deprecate where users live, remove where alternatives exist) is right and the new override-detection helper is the load-bearing piece that fixes #20120's regression. Three nits to address pre-merge: (a) test for the `uses_legacy_sandbox_mode_override = true` branch, (b) release-note for the `pub fn create_sandbox_mode` deletion, (c) verify and document the `--full-auto` → `on-request` semantic in `run_in_container.sh`.

## What I learned

This is a textbook "stage the deprecation, then walk back the over-aggressive deletion" sequence. #20117 added the replacement (`--permissions-profile`). #20119 added the second replacement (`--replay-json`). #20120 deleted `--full-auto` everywhere on the assumption coverage was complete. This PR (#20133) fixes the gap: `--full-auto` was *also* the way to opt into a writable legacy `sandbox_mode` config from the bare `codex sandbox <os>` command (since legacy configs default-to-read-only there), and the `-c sandbox_mode=...` override path needed to be wired before deletion was safe. Lesson: when removing a flag that defaults a config field, audit the "what was the only way to reach this config state from this entry point" question before merge.
