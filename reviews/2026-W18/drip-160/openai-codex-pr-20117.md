# openai/codex PR #20117 — feat(cli): add explicit sandbox permission profiles

- Repo: `openai/codex`
- PR: https://github.com/openai/codex/pull/20117
- Head SHA: `7b4615b040f91c935ca67b27e38ead2de3736388`
- State: OPEN, +131/-17 across 3 files

## What it does

Adds a new `--permissions-profile <NAME>` CLI flag to all three `codex sandbox <macos|linux|windows>` debug subcommands, lets the operator override the active configuration's default permission profile at the *debug-sandbox* layer without a config file edit. Internally the flag is plumbed by appending a synthetic `cli_overrides` entry `("default_permissions", TomlValue::String(name))` inside `load_debug_sandbox_config_with_codex_home` *before* the rest of the override stack is folded in. As a side cleanup the diff also restructures the per-OS `SandboxType` enum so seatbelt-only state (`allow_unix_sockets`, `log_denials`) lives inside the `Seatbelt { ... }` variant rather than being passed down as separate parallel arguments — a representational sharpening that prevents the "log_denials passed as `false` for Landlock/Windows" awkwardness.

## Specific reads

- `cli/src/lib.rs:25-32, 52-59, 66-77` — the same three lines repeated across `SeatbeltCommand`, `LandlockCommand`, `WindowsCommand`:
  ```rust
  /// Named permissions profile to apply from the active configuration stack.
  #[arg(long = "permissions-profile", value_name = "NAME")]
  pub permissions_profile: Option<String>,
  ```
  Three identical blocks deserve a `#[derive(Args)]`-style shared struct or at minimum a comment that they must stay in lockstep — silent drift between the three (e.g. one accepting `--permission-profile` singular by mistake) would split clients.

- `cli/src/debug_sandbox.rs:609-617` — the load-bearing override-injection:
  ```rust
  if let Some(permissions_profile) = permissions_profile {
      cli_overrides.push((
          "default_permissions".to_string(),
          TomlValue::String(permissions_profile),
      ));
  }
  ```
  Pushed onto `cli_overrides` *before* `build_debug_sandbox_config(cli_overrides.clone(), ...)` is called, so the synthetic override participates in the same precedence stack as user-supplied `-c k=v` overrides. The position-at-end gives it lowest precedence relative to other CLI overrides (later entries win in the typical push-then-fold pattern, but this depends on how `build_debug_sandbox_config` handles dupes — not visible in this diff). If a user passes both `--permissions-profile foo` *and* `-c default_permissions=bar`, the resolution order should be documented or pinned by a test; today neither is present.

- `cli/src/debug_sandbox.rs:80-89` — the `SandboxType` restructuring:
  ```rust
  enum SandboxType {
      #[cfg(target_os = "macos")]
      Seatbelt {
          allow_unix_sockets: Vec<AbsolutePathBuf>,
          log_denials: bool,
      },
      Landlock,
      Windows,
  }
  ```
  This is the right shape — it makes invalid states (`Landlock` with `allow_unix_sockets`) representationally impossible. The `#[cfg_attr(not(target_os = "macos"), allow(unused_variables))]` workaround on the old function signature is gone with it. The match at line 167-170 (`SandboxType::Seatbelt { log_denials, .. } => log_denials.then(DenialLogger::new).flatten()`) is clean.

- Test coverage at `debug_sandbox.rs:783-844` is solid: `debug_sandbox_honors_explicit_builtin_permission_profile` pins `:workspace` against the canonical `PermissionProfile::workspace_write().file_system_sandbox_policy()`, and `debug_sandbox_honors_explicit_named_permission_profile` pins a custom-named profile against an equivalent direct-config path. The matching CLI parse test at `main.rs:1929-1950` covers only the macos variant — landlock and windows variants are untested at the parse layer.

## Verdict: `merge-after-nits`

## Rationale

The feature is well-shaped: a single named knob that composes with the existing override stack rather than introducing a parallel sandbox-config concept, and the seatbelt-state-into-variant cleanup is the right call. Three nits to address before merge: (a) the three identical `permissions_profile: Option<String>` blocks across the per-OS clap structs should either share a derived struct or carry a `// keep in sync` comment; (b) the parse test at `main.rs:1929-1950` should be repeated for `linux` and `windows` subcommands so a future flag-rename doesn't silently drop two-thirds of platforms; and (c) the precedence interaction between `--permissions-profile foo` and `-c default_permissions=bar` should either be pinned by a test or documented in the doc comment of the new flag — today the `cli_overrides.push(...)` ordering implies last-write-wins, but that's an implementation detail of `build_debug_sandbox_config` not visible in this diff.
