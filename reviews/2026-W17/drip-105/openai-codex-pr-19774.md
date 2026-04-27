---
pr: https://github.com/openai/codex/pull/19774
sha: cac068aa
diff: +192/-161
state: OPEN
---

## Summary

Continues the multi-PR permissions-stack migration (siblings #19772, #19709, #19776) by making `SessionConfiguredEvent` profile-only on the wire: removes the legacy `sandbox_policy: SandboxPolicy` field and changes `permission_profile` from `Option<PermissionProfile>` to a non-optional `PermissionProfile`. Adds a custom `Deserialize` impl that accepts the legacy shape (with `sandbox_policy`, no `permission_profile`) and lifts it into a profile via `PermissionProfile::from_legacy_sandbox_policy_for_cwd`, so older emitters and on-disk session logs keep working.

## Specific observations

- `codex-rs/protocol/src/protocol.rs:3536-3611` — the field swap is the headline change: `sandbox_policy` is gone, `permission_profile` is `PermissionProfile` (no `Option`, no `default`). Combined with the custom deserializer, this enforces "every session has a profile" at the type level for new producers while preserving back-compat for old wire payloads. Correct shape.
- `codex-rs/protocol/src/protocol.rs:3638-3697` — the manual `Deserialize` impl uses a private `Wire` struct with both `sandbox_policy: Option<SandboxPolicy>` and `permission_profile: Option<PermissionProfile>`, then resolves with the right precedence:
  ```rust
  let permission_profile = match (wire.permission_profile, wire.sandbox_policy) {
      (Some(permission_profile), _) => permission_profile,
      (None, Some(sandbox_policy)) => PermissionProfile::from_legacy_sandbox_policy_for_cwd(
          &sandbox_policy,
          wire.cwd.as_path(),
      ),
      (None, None) => return Err(serde::de::Error::missing_field("permission_profile")),
  };
  ```
  The `(Some, _)` arm explicitly discards a coexisting `sandbox_policy`, which is the right call: when both are present, the profile is canonical. The `(None, None)` arm returns a missing-field error naming `permission_profile`, which gives a clean error message for new payloads but is slightly misleading for genuinely-old payloads that happen to be missing both — those will report "missing permission_profile" when the user might be debugging "why isn't my old sandbox_policy being read?". A two-field error message ("missing permission_profile (and no legacy sandbox_policy fallback)") would be friendlier.
- `codex-rs/protocol/src/protocol.rs:5184-5206` — the `deserialize_legacy_session_configured_event_uses_sandbox_policy` test pins exactly the migration contract: legacy payload with only `sandbox_policy: read-only` must deserialize to `PermissionProfile::read_only()`. That's the right pin. Worth adding a second test for the `(Some, Some)` case to lock in "profile wins, sandbox_policy is silently discarded."
- `codex-rs/core/src/session/session.rs:867-880` — the emitter side now ships only `permission_profile`. The old code computed `session_sandbox_policy` and shipped both; now `session_configuration.permission_profile()` is the single source. Note the change from `Some(session_configuration.permission_profile())` to `session_configuration.permission_profile()` — confirms the field is no longer optional. Clean.
- `codex-rs/exec/src/lib.rs:1043-1049` — `session_configured_from_thread_response` now reconstructs the profile when receiving a thread-resume response that doesn't carry one:
  ```rust
  permission_profile: permission_profile.unwrap_or_else(|| {
      PermissionProfile::from_legacy_sandbox_policy_for_cwd(&sandbox_policy, cwd.as_path())
  }),
  ```
  Same fallback as the deserializer. But this site receives an `Option` from `ThreadResumeResponse`, which means the *response* type still carries an optional profile. Good — that's the back-compat surface for a server that hasn't been upgraded. Just verify the `app_server_protocol` types match this expectation; otherwise the `unwrap_or_else` is dead code.
- `codex-rs/app-server/src/codex_message_processor.rs:4674-4700` and `:5405-5430` — both call sites now bind `let config_snapshot = ...await;` once and reuse it for both `permission_profile` and `sandbox`. Previously these were two awaits + an implicit move. Cleaner and avoids the `await` race where the two reads could see different config snapshots. Notice `sandbox: config_snapshot.sandbox_policy.into()` is still pulled from the snapshot rather than from `session_configured.sandbox_policy` (which no longer exists) — correct.
- `codex-rs/mcp-server/src/outgoing_message.rs +6/-12` and `event_processor_with_json_output.rs +3/-4` — test fixtures updated to construct `permission_profile: PermissionProfile::read_only()` and drop `sandbox_policy`. Mechanical, no behavior change. The serialized-output assertion at `:387-394` swaps `"sandbox_policy": {"type": "read-only"}` for `"permission_profile": session_configured_event.permission_profile` — i.e. uses the actual emitted value rather than a hardcoded shape. Loose but acceptable since the field is the test subject.
- The new `Wire` struct duplicates ~16 fields of the public type. If the field set drifts, the `Wire` will silently drop new fields on legacy-payload deserialization. A `#[serde(other)]` catch-all isn't possible in derive `Deserialize`, but a comment at the top of `Wire` warning "keep in sync with `SessionConfiguredEvent`" would help; alternatively, a single `#[derive(Deserialize)]` on a wrapper that contains the public struct flattened plus the legacy `sandbox_policy: Option<SandboxPolicy>` would avoid the duplication entirely (though it requires `#[serde(flatten)]` discipline).

## Verdict

`merge-after-nits` — disciplined wire-format migration with the right back-compat contract: type-level enforcement for new producers, lossless lift for legacy payloads. Three small follow-ups: (1) friendlier error message in the `(None, None)` arm naming both fields, (2) one additional test pinning that `permission_profile` wins when both fields are present, (3) maintenance comment on the `Wire` struct or a `#[serde(flatten)]` refactor to avoid silent field-drift.

## What I learned

When you need to remove an optional field from a wire format but support legacy emitters indefinitely, splitting `Serialize` (auto-derive, only emits the new field) from a manual `Deserialize` (accepts both shapes, resolves to the new one) gives you "only one canonical shape on the wire going forward, but old payloads still load." The shape that gets serialized and the shape that gets deserialized don't have to be the same — and pretending they must is what forces people to keep `Option<T>` forever.
