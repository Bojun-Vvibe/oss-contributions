# openai/codex #20015 — core tests: configure profiles directly

- PR: https://github.com/openai/codex/pull/20015
- Head SHA: `95f520d74f5b3f79473a2a83c1750692ea314dfa`
- Author: bolinfest
- Files touched: `codex-rs/core/tests/suite/code_mode.rs`, `codex-rs/core/tests/suite/codex_delegate.rs`, `codex-rs/core/tests/suite/otel.rs`, `codex-rs/core/tests/suite/tools.rs`

## Observations

- This is the tail-end of a stack (#20008/#20010/#20011/#20013 land first) that migrates test scaffolding from the legacy `SandboxPolicy` enum to the new `PermissionProfile` model. The diff swaps `use codex_protocol::protocol::SandboxPolicy;` for `use codex_protocol::models::PermissionProfile;` across 4 test suite files.
- `codex-rs/core/tests/suite/code_mode.rs:2611-2627` — replaces the inline `SandboxPolicy::DangerFullAccess` + `permission_profile: None` construction with `let (sandbox_policy, permission_profile) = turn_permission_fields(PermissionProfile::Disabled, cwd.as_path());`. This goes through the canonical helper `turn_permission_fields` rather than hand-constructing the tuple, which is the right direction.
- `codex-rs/core/tests/suite/codex_delegate.rs:67-69` and `:152-154` — switches `set_legacy_sandbox_policy(SandboxPolicy::new_read_only_policy())` to `permissions.set_permission_profile(PermissionProfile::read_only())`. Method names line up with non-test call-sites, so future refactors won't have to special-case test code.
- Pure test refactor; no runtime code changes. Risk is contained to test-suite drift if the rest of the stack lands out of order.

## Verdict: `merge-as-is`

**Rationale:** Mechanical migration that keeps tests honest by configuring profiles the same way production code does. Once the precursors in the stack land, this should go in without further review. The `Constrained::allow_any` pattern around it is consistent with other tests in the same suite.
