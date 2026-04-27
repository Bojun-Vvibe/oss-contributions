---
pr: https://github.com/openai/codex/pull/19772
sha: c188a22f
diff: +377/-175
state: OPEN
---

## Summary

Continues the multi-PR permissions migration by making `ConfigToml`'s default-resolution path produce a `PermissionProfile` directly rather than building a legacy `SandboxPolicy` first and converting back. Adds `PermissionProfile::workspace_write_with()` and `FileSystemSandboxPolicy::legacy_workspace_write()` helpers, switches `ConfigToml::derive_sandbox_policy` to a thin compatibility wrapper around the new `derive_permission_profile`, and threads cwd-aware legacy-workspace-write defaults through `Config::load()` so existing metadata carveouts (`.git`, `.agents`) keep working.

## Specific observations

- `codex-rs/config/src/config_toml.rs:644-660` — `derive_permission_profile` is the new canonical entry point; the function flips return type from `SandboxPolicy` to `PermissionProfile` and the `WorkspaceWrite` arm now constructs `PermissionProfile::workspace_write_with(writable_roots, network_policy, exclude_tmpdir_env_var, exclude_slash_tmp)` where the boolean `network_access` from the TOML is widened to a `NetworkSandboxPolicy::{Enabled,Restricted}` enum at `:692-696`. Correct call shape — the boolean is positionally ambiguous so the explicit enum is the right ergonomic upgrade.
- `:678-681` — the Windows downgrade-if-unsupported check is hoisted out of the closure (which `:707` previously called via `downgrade_workspace_write_if_unsupported(&mut sandbox_policy)`) and into a flat `let workspace_write_unsupported = ...` — same predicate, less indirection. The `WindowsSandboxLevel::Disabled` carveout (don't downgrade if the experimental Windows sandbox is opted-in) is preserved verbatim.
- `codex-rs/protocol/src/permissions.rs +178/-65` is the largest hunk and where the substantive new API lives — `PermissionProfile::workspace_write_with(...)` is the public constructor that this PR introduces and the rest of the codebase migrates to. Without seeing those lines I cannot pin exact behavior, but the PR description's verification list (`workspace_write --lib`, `legacy_workspace_write_projection --lib`, `with_additional_legacy_workspace_writable_roots_protects_metadata --lib`) names three test families and the third one specifically pins the metadata-carveout behavior the PR claims to preserve.
- `codex-rs/core/src/config/config_tests.rs +79/-15` — coverage delta is asymmetric (5x more added than removed) suggesting genuinely new pin tests rather than a churn-rewrite. The "missing `.git` and `.agents` remain writable for legacy compatibility" claim in the PR body is a real footgun fix: making them read-only when absent would break `git init` and `agents init` flows that legitimately create those directories at runtime.
- `derive_sandbox_policy` is kept as a wrapper — good migration discipline, lets downstream callers move at their own pace — but no `#[deprecated]` attribute on the wrapper means there is no compile-time signal to stop adding new call sites. A `#[deprecated(note = "use derive_permission_profile and project the legacy policy at the boundary")]` would actively shrink the migration surface.

## Verdict

`merge-after-nits` — sound migration step with the right contract (push profile-native resolution upstream, keep legacy projection at compatibility boundaries) and disciplined preservation of cwd-aware metadata defaults. Two minor follow-ups: (1) `#[deprecated]` on `derive_sandbox_policy` to surface remaining migration debt in `cargo build`, (2) one assertion that pins the boolean-to-enum widening at `:692-696` doesn't silently flip semantics for the `network_access: false` default case (the test list mentions `derive_sandbox_policy` coverage but not the network-policy-enum path explicitly).
