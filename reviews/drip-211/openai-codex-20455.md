# openai/codex#20455 — windows-sandbox: accept permission profiles from callers

- PR: https://github.com/openai/codex/pull/20455
- Head SHA: `f63f05f8a6895f34a9938e8ea25789780cc24f4e`
- Author: bolinfest (Michael Bolin)
- Files: `Cargo.lock` −1, `core/src/windows_sandbox.rs` +45/−19,
  `core/src/windows_sandbox_read_grants.rs` +1/−9, `tui/Cargo.toml` −2,
  `tui/src/app/event_dispatch.rs` +2/−10, `tui/src/app/platform_actions.rs`
  +3/−7, `tui/src/chatwidget.rs` +3/−6
- Sapling stack: sits below #20456

## Context

The TUI was constructing `SandboxPolicy` values via the legacy
compatibility projection just to feed the Windows-sandbox setup
helpers (elevated setup, legacy preflight, world-writable scan,
read-root refresh). That meant every TUI call site had to import
both `PermissionProfile` *and* `SandboxPolicy`, do the projection,
and pass the legacy type across the core boundary — which kept
`codex-windows-sandbox` as a direct TUI dependency
(`tui/Cargo.toml` line removed in this PR) for the sole purpose
of speaking `SandboxPolicy`.

## Design

The PR moves the boundary one layer in: core's
`windows_sandbox.rs` now exports four `pub fn` shells that take
`&PermissionProfile` and do the legacy projection internally
via a private helper at `windows_sandbox.rs:60-72`:

```rust
fn compatibility_windows_sandbox_policy(
    permission_profile: &PermissionProfile,
    policy_cwd: &Path,
) -> SandboxPolicy {
    let file_system_policy = permission_profile.file_system_sandbox_policy();
    compatibility_sandbox_policy_for_permission_profile(
        permission_profile,
        &file_system_policy,
        permission_profile.network_sandbox_policy(),
        policy_cwd,
    )
}
```

The four call-site rewrites at `:187-265`:
- `run_elevated_setup` — `&SandboxPolicy` → `&PermissionProfile`
- `run_legacy_setup_preflight` — same
- `run_setup_refresh_with_extra_read_roots` — same
- `apply_world_writable_scan_and_denies` — new core wrapper that
  takes profile and forwards a derived policy

The non-Windows `cfg`-gated stubs are renamed in lockstep
(`_policy: &SandboxPolicy` → `_permission_profile:
&PermissionProfile`) so the cross-platform build stays warning-
free.

`windows_sandbox_read_grants.rs` loses its own duplicate
projection (`+1/−9`) because the projection now lives at one
canonical site (the helper). TUI side
(`event_dispatch.rs`, `platform_actions.rs`, `chatwidget.rs`)
goes from `let policy = compat_project(profile); call(&policy,
...)` to direct `call(&profile, ...)` — net `−10/−7/−6` lines
across the three files.

`Cargo.lock` and `tui/Cargo.toml` drop the
`codex-windows-sandbox` direct dep on TUI — TUI no longer
speaks the legacy type at all on this boundary.

## Risks / nits

- The helper at `:60-72` is called from each of the four
  `pub fn` shells, which means on Windows the projection runs
  on every elevated-setup / preflight / refresh call. The
  projection is cheap (a few struct copies), but if any caller
  invokes these helpers in a tight loop the per-call cost is
  worth a `#[cfg(target_os="windows")]` benchmark. Probably a
  non-issue but worth a sanity check.
- The non-Windows stub of `run_elevated_setup` now takes
  `_permission_profile: &PermissionProfile` and discards it.
  `_policy_cwd: &Path` etc. all underscore-prefixed. Fine,
  but a one-line `unreachable!()` or `panic!("not supported on
  non-Windows")` would be more honest than the current
  silent `Ok(())` shape — though I see the current behavior
  matches the pre-PR stub, so this is a pre-existing artifact
  not introduced here.
- `windows_sandbox_read_grants.rs` `+1/−9` removes its own
  projection but the diff slice didn't show what `+1` is. Worth
  confirming it's just the import-line cleanup.

## Verdict

**`merge-as-is`**

Textbook boundary-relocation refactor. The legacy
`SandboxPolicy` projection now lives at exactly one site
(the helper at `windows_sandbox.rs:60-72`), TUI loses its
direct `codex-windows-sandbox` dependency, three TUI files
go simpler by ~7 lines each, and the four `pub fn` shells
present a uniform `&PermissionProfile` boundary to all
callers. Verified by `cargo check -p codex-core -p codex-tui
--tests` and `just fix` per PR description. Continues the
broader migration that #20456 (the next PR up the stack)
also serves.

## What I learned

When a half-migrated abstraction has N callers each doing
their own projection, the right move is to push the
projection one layer toward the consumer (here: into core's
own `windows_sandbox.rs`) so the projection happens in one
place that knows the legacy type, and every TUI/UI/etc.
caller speaks only the new type. The moment you can drop a
direct dependency from a layer (TUI no longer needs
`codex-windows-sandbox`), you know the boundary has
relocated cleanly.
