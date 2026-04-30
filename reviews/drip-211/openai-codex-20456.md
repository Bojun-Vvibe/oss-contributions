# openai/codex#20456 — config: gate legacy writable roots from profiles

- PR: https://github.com/openai/codex/pull/20456
- Head SHA: `93484640e4cd56ededef0161c199212b5fe768d6`
- Author: bolinfest (Michael Bolin)
- Files: `core/src/config/mod.rs` +24/−15
- Sapling stack tip; sits above #20455, #20452, etc.

## Context

`Config::load_config_with_layer_stack` had two near-identical
detours that built a *throwaway* `SandboxPolicy` from the active
`PermissionProfile` solely to ask "is the result `WorkspaceWrite`?"
before applying legacy `writable_roots` and memory-root extras.
That kept the legacy sandbox-policy abstraction alive in the
middle of config loading even though every other consumer in
core had already migrated to `PermissionProfile` as the canonical
runtime model. A third site at `:2325-2336` already did the
"right" check (matches!(enforcement, Managed) + can_write_path +
!has_full_disk_write) but inline.

## Design

The PR introduces one helper at `config/mod.rs:1701-1713`:

```rust
fn accepts_legacy_additional_writable_roots(
    permission_profile: &PermissionProfile,
    file_system_sandbox_policy: &FileSystemSandboxPolicy,
    cwd: &Path,
) -> bool {
    matches!(permission_profile.enforcement(), SandboxEnforcement::Managed)
        && file_system_sandbox_policy.can_write_path_with_cwd(cwd, cwd)
        && !file_system_sandbox_policy.has_full_disk_write_access()
}
```

The three predicates capture exactly the invariant the inline
code at `:2325` already encoded:

1. **Managed enforcement only** — disabled / external / read-only
   profiles must never silently grow extra writable roots.
2. **Already grants write to the active workspace** — extending
   write access to a profile that doesn't write the cwd would
   be a *grant*, not an *extension*; the legacy compat surface
   is for additive-only behavior.
3. **Not full-disk** — full-disk profiles writing more is a
   no-op-or-confusing escalation; skip.

Both `:2181` and `:2234` sites are converted from the
`compatibility_sandbox_policy_for_permission_profile(...) →
matches!(SandboxPolicy::WorkspaceWrite { .. })` round-trip to a
direct `accepts_legacy_additional_writable_roots(...)` call.
Net wire/runtime behavior is identical — same set of profiles
gets the legacy extras, none added or removed — but the
compatibility-projection now happens in only one logical place
(the named helper at the top of the file) instead of inline at
two sites with implicit dependency on `WorkspaceWrite` being
the discriminant.

## Risks / nits

- The third inline site at `:2325-2336` (the explicit-extra-
  roots branch) is also rewritten to call the helper, which
  is correct and removes the third copy. Worth confirming
  `cargo test -p codex-core workspace_profile` (called out in
  the PR's verification list) actually exercises all three
  conversion sites — a `#[test]` matrix over (profile,
  has_extra_roots, has_memory_roots) ensuring the legacy
  extras land in the same set of files as before would lock
  this in.
- The helper takes `&PermissionProfile, &FileSystemSandboxPolicy,
  &Path` — three args, all needed, but the `FileSystemSandboxPolicy`
  is derivable from the profile via `permission_profile
  .file_system_sandbox_policy()`. Two of the three callers
  pass a `file_system_sandbox_policy` that has *already been
  extended* with extras from prior config layers, so the
  caller-passed value is load-bearing (it's not equal to
  `permission_profile.file_system_sandbox_policy()` at the
  call site). One-line comment naming this would prevent a
  future "simplification" that drops the parameter.
- The `legacy_sandbox_policy()` compatibility projection on
  `Config` is intentionally left in place — the PR description
  notes call sites that still need to speak the old API. That's
  the right scope; trying to remove it in this PR would balloon
  the diff into the consumer side.

## Verdict

**`merge-as-is`**

Pure refactor — three callers now share one named predicate,
the predicate body is byte-identical to the existing inline
check at `:2325`, and the `compatibility_sandbox_policy_for_
permission_profile(...)` round-trip detour is gone from two
of the three sites. No behavior change beyond making the
"who gets legacy extras?" decision locally enforceable from
one place. Sits cleanly at the tip of a 67-PR Sapling stack
that has been systematically draining `SandboxPolicy` from
internal-construction paths.

## What I learned

The "build a throwaway value of an old type to ask one
boolean question about it" anti-pattern is one of the
clearest tells that a migration is half-done. The fix isn't
to delete the old type (other surfaces still need it for
external-protocol compat), it's to extract the specific
boolean question into a named predicate and let the old
type be answered-about only at its own boundary.
