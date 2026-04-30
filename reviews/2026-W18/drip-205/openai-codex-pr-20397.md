# Review: openai/codex#20397 — app-server-test-client: select permission profiles by name

- PR: https://github.com/openai/codex/pull/20397
- Author: bolinfest (Michael Bolin)
- headRefOid: `c210b12f392d35b28a667365742d66c6bee048af`
- Files: `codex-rs/app-server-test-client/src/lib.rs` (+19/-16)
- Verdict: **merge-after-nits**

## Analysis

Migrates the manual app-server test client off the legacy `SandboxPolicy` turn override and onto
`PermissionProfileSelectionParams::Profile { id, modifications: None }` via a new local helper
`select_permission_profile(&str)` (lib.rs:623-628). The call sites that previously synthesized
`SandboxPolicy::ReadOnly { network_access: false }` or `SandboxPolicy::DangerFullAccess` now request
the named profiles `:read-only` and `:danger-no-sandbox` respectively. This matches the v2 turn
request contract and keeps debug/manual harnesses aligned with the public app-server API direction.

The `SendMessagePolicies` struct's `sandbox_policy: Option<SandboxPolicy>` field becomes
`permission_profile_id: Option<&'static str>` (lib.rs:632-637). The `'static str` lifetime is the
right choice for this internal harness — every concrete call site (`trigger_cmd_approval` at
lib.rs:887, `trigger_patch_approval` at lib.rs:911, `no_trigger_cmd_approval` at lib.rs:932, the
`send_message`/`send_message_v2_endpoint` helpers around lib.rs:649/692) passes a string literal.
The mapping back to selection params happens once inside `send_message_v2_with_policies`
(lib.rs:968-973) via `policies.permission_profile_id.map(select_permission_profile)`.

**Nit (non-blocking):** `select_permission_profile` always passes `modifications: None`, which is
fine for the current callers but loses the modifications surface entirely. If a future test ever
needs to select a profile with modifications it'll have to bypass the helper. A small ergonomic
improvement would be `fn select_permission_profile(id: &str, modifications: Option<...>)` or a
second `select_permission_profile_with_mods` — but I wouldn't block on it; YAGNI is the right call
for now.

**Nit (non-blocking):** `live_elicitation_timeout_pause` (lib.rs:1263-1266) now requests
`:danger-no-sandbox` by name instead of `SandboxPolicy::DangerFullAccess`. Worth confirming that
the built-in profile id `:danger-no-sandbox` is guaranteed-present on the server side under the
default profile registry — if it can be removed/renamed, this elicitation flow silently breaks.
A short comment pointing to the canonical profile-id source of truth would help future readers.

Otherwise clean. Merge after either of those nits is addressed (or just merged as-is and follow-up
later).
