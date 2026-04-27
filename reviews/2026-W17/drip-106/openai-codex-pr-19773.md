# openai/codex PR #19773 — permissions: require profiles in TUI thread state

- **Link**: https://github.com/openai/codex/pull/19773
- **Head SHA**: `c543c8d68160e5d2b5a5e16e21226c50c0955fed`
- **Author**: bolinfest
- **Size**: +128 / -35 across 5 files
- **Verdict**: `merge-after-nits`

## Files changed
- `codex-rs/tui/src/app/tests.rs` — pin tests updated to drop `Some(...)` wrappers around `permission_profile`; the previously-`None` assertion in `thread_read_session_state_does_not_reuse_primary_permission_profile` is rewritten to expect a freshly-derived legacy profile bound to the *read thread's* cwd.
- `codex-rs/tui/src/app/thread_events.rs` — test helper updated to the new non-optional shape.
- `codex-rs/tui/src/app/thread_session_state.rs` — `permission_profile` becomes `PermissionProfile` (not `Option<PermissionProfile>`); two new helpers `active_legacy_permission_profile_for_cwd` / `active_legacy_sandbox_policy_for_cwd` synthesize a profile from the local legacy `SandboxPolicy` against the *target thread's* cwd.
- `codex-rs/tui/src/app_server_session.rs` — wire-format compat: `started.session.permission_profile` now `.expect("response includes profile")`.
- `codex-rs/tui/src/chatwidget.rs:1618-1622` — `thread_session_state_to_legacy_event` re-wraps the now-required field in `Some(...)` for the legacy event boundary.

## Analysis

This is the third installment in the in-flight permissions migration (siblings #19772 already merged, #19774 / #19776 in flight that drip-105's #19774 review also covered). The contract being established is "the TUI's cached `ThreadSessionState` must always carry a `PermissionProfile`; legacy `sandbox_policy`-only payloads get lifted to a profile at the boundary, and the cached value is never optional thereafter."

The interesting call site is `thread_session_state.rs` lines around 49-94: previously, when materializing a `ThreadSessionState` for a thread/read response, the code set `permission_profile: None` because the server's thread/read response carries no authoritative profile. The fix correctly observes that *None is the wrong representation* — downstream consumers need *some* profile, and the right profile to fall back to is one derived from the local legacy `SandboxPolicy` against the **read thread's cwd**, not the primary session cwd. The new helper `active_legacy_permission_profile_for_cwd` does exactly that: `PermissionProfile::from_legacy_sandbox_policy_for_cwd(&sandbox_policy, cwd)` where `sandbox_policy` is itself resolved against the target cwd.

The pin test rewrite at `tests.rs:2858-2871` is the right shape — it asserts the rebuilt-from-legacy profile rather than asserting `None`, with a comment that explains the policy ("reusing the primary session profile would reinterpret cwd-bound entries against the read thread cwd"). That comment is load-bearing — without it, a future contributor will assume the change "just" made the field non-optional and might short-circuit the `_for_cwd` derivation.

The `chatwidget.rs:1621` `permission_profile: Some(session.permission_profile)` re-wrap at the legacy event boundary is the unavoidable cost of this migration: the wire `SessionConfiguredEvent` still carries `Option<PermissionProfile>` (sibling #19774 makes that profile-only — see drip-105 review). After both land, this `Some(...)` should disappear.

The `app_server_session.rs` change to `.expect("response includes profile")` is the **one risk site**: it asserts that any started-session response from a current app-server includes a profile. If this PR ships before app-server #19774's profile-only `SessionConfigured` ships, an old-binary app-server still emitting profile-less SessionConfigured will now panic the TUI rather than fall back. The PR's `## Why` section calls out the back-compat concern in prose ("Those must still hydrate correctly without reinterpreting cwd-bound grants later") but the `.expect()` here is in tension with that — it's only safe because there's a separate path for the started response.

## Nits to address before merge

1. **Justify the `.expect("response includes profile")` at `app_server_session.rs` test site** with a comment naming which app-server version guarantees profile presence, or downgrade to `.unwrap_or_else(...)` with a fallback profile derivation similar to the read-thread path. As written, a stale app-server during a rolling upgrade will TUI-panic rather than degrade.
2. **The two new helpers `active_legacy_permission_profile_for_cwd` / `active_legacy_sandbox_policy_for_cwd` are private to `App`** but logically belong on `Permissions` itself — consider lifting once the migration settles, since `chat_widget.config_ref().permissions` is fetched both times and the helpers do nothing the `Permissions` type couldn't expose directly.
3. **No test pins the cwd-mismatch scenario directly**: a thread/read response where `thread.cwd != self.config.cwd` should return a profile whose grants are bound to `thread.cwd`. The existing `thread_read_session_state_does_not_reuse_primary_permission_profile` test asserts the profile differs from the primary's, but doesn't directly assert which cwd the resulting grants are bound to. A one-line `assert_eq!(session.permission_profile.cwd_grants(), expected_grants_for(thread.cwd))` (or whatever the accessor is) would lock the load-bearing semantic.
4. The PR description doesn't enumerate which sibling PR ships the profile-only `SessionConfigured`. Add `Depends on #19774` or note the order constraint so the merge-train operator doesn't land this first.

## What I learned

The "drop `Option`, derive a sensible default at every materialization site" migration pattern is common but the *cwd-binding* discipline this PR captures is the part that's easy to get wrong: when you derive a profile from local legacy settings, you must derive it against the *target* artifact's cwd, not the global session cwd, or you'll reinterpret workspace-write grants against the wrong root. The pin test that names this trap explicitly is the right kind of regression scaffolding.
