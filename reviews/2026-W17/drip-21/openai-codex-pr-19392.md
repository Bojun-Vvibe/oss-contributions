# openai/codex#19392 — permissions: derive compatibility policies from profiles

- **PR:** https://github.com/openai/codex/pull/19392
- **Head SHA:** `e8e99026399e23b4b2a4785167a11dd4cb1bcb80`
- **Files:** 39 files changed (+494/-411). Stack position: between #19391
  (runtime-config-profile-backed) and #19393 (approval/sandbox consumers).
- **Verdict:** **merge-after-nits**

## Context

Mid-stack PR in the long-running permission-profile migration sweep
that this index has been tracking since drip-17 (#19395, #19394,
#19393, #19414). The runtime is now profile-backed (#19391); legacy
APIs, persisted thread metadata, and OS sandbox adapters still need
`SandboxPolicy` values, and prior to this PR each of those callers
was reaching back into the legacy `permissions.sandbox_policy` field
on `Config`. This PR centralizes those projections so that legacy
shapes are *derived* from `PermissionProfile` rather than read from
parallel state.

## Problem being fixed

After #19391 there are *two* parallel sources of truth on `Config`:
the runtime `PermissionProfile` and the legacy `SandboxPolicy`. Every
caller that wanted "the legacy policy" was free to read either field
directly, and the two fields would drift the moment a code path
mutated the profile but not the legacy mirror. The fix is to make
`SandboxPolicy` (and `FileSystemSandboxPolicy` /
`NetworkSandboxPolicy`) **derived** from the profile, not stored
separately, so there's exactly one writer and many derived readers.

## Design — what changed

Three concentric rings of change:

1. **Compat helpers on `PermissionProfile`** — new richer projections
   in `protocol/src/permissions.rs` (+66/-3) that produce split
   `(FileSystemSandboxPolicy, NetworkSandboxPolicy)` pairs preserving
   profile-specific filesystem details (e.g. `denyRead` paths,
   per-cwd writability) instead of round-tripping through the lossy
   single `SandboxPolicy` enum.

2. **Replace direct field reads with method calls.** Visible in
   `app-server/src/codex_message_processor.rs` and
   `cli/src/debug_sandbox.rs`:

   ```rust
   // before
   &self.config.permissions.file_system_sandbox_policy,
   self.config.permissions.network_sandbox_policy,

   // after
   let configured_file_system_sandbox_policy =
       self.config.permissions.file_system_sandbox_policy();
   ...
   ```

   Same swap in test code (`debug_sandbox.rs:715-732` test block):
   four `permissions.file_system_sandbox_policy` field accesses
   become method calls. This is a mechanical sweep with consistent
   shape across every consumer.

3. **MCP elicitation auto-approval rewritten in profile terms** —
   the most semantically interesting hunk, in
   `codex-mcp/src/mcp/mod.rs:89-110`:

   ```rust
   // before: pattern-match the legacy enum
   approval_policy == AskForApproval::Never
       && matches!(
           sandbox_policy,
           SandboxPolicy::DangerFullAccess | SandboxPolicy::ExternalSandbox { .. }
       )

   // after: ask the profile
   match permission_profile {
       PermissionProfile::Disabled
       | PermissionProfile::External { .. } => true,
       PermissionProfile::Managed { file_system, .. } => {
           file_system.to_sandbox_policy().has_full_disk_write_access()
       }
   }
   ```

   Notably, the new branch *includes* `Managed` profiles whose
   filesystem permissions resolve to full-disk-write access — the
   old enum-only matcher silently excluded those cases, because
   `Managed { full_disk_write }` is a strictly newer shape than
   what `SandboxPolicy` could express. So this is also a behavior
   change, not just a refactor: there's a previously-unreachable
   auto-approve case that now triggers.

## Test coverage

`codex-mcp/src/mcp/mod_tests.rs:64-95` adds explicit coverage for
the three relevant cases:

- `Managed { Unrestricted, Enabled }` + `Never` ⇒ auto-approved ✅
- `Managed { Unrestricted, Restricted }` + `Never` ⇒ auto-approved ✅
- read-only legacy-derived profile + `Never` ⇒ NOT auto-approved ✅
- `Managed { Unrestricted, Enabled }` + `OnRequest` ⇒ NOT auto-approved ✅

Good — covers the new behavior change and pins the negative case.
The "external" auto-approval branch isn't directly tested but is
trivially structural.

## Strengths

- **Reduces the number of places that can desync.** Pre-PR, ~12
  call sites read the legacy field directly; post-PR they all funnel
  through `permissions.file_system_sandbox_policy()` /
  `network_sandbox_policy()`, which is the right shape if the field
  ever needs to become lazy or computed.
- **Test for the MCP auto-approval semantic change is explicit.**
  The four-case matrix locks in the intended new behavior so
  future refactors can't silently re-narrow it.
- **Small, focused diffs in 39 files** — each individual change is
  one of three patterns (field→method swap; legacy-policy→profile
  signature change; new derived helper). Easy to review.

## Risks / nits

1. **Behavior change is buried.** Auto-approving MCP elicitations
   for the previously-unreachable `Managed { Unrestricted, Enabled }`
   case is a real semantic widening of "auto-approve". This deserves
   one explicit line in the PR body — currently the body says
   *"compatibility derives from `PermissionProfile`"*, which a
   reviewer can read as a pure refactor. It is not.
2. **`PermissionProfile::default()`** is being passed to
   `read_mcp_resource`, `collect_mcp_snapshot_with_detail`, and
   `collect_mcp_server_status_snapshot_with_detail` (replacing
   `SandboxPolicy::new_read_only_policy()`). Whether `default()`
   round-trips to the same effective auto-approval semantics as
   `new_read_only_policy()` deserves a single assertion test —
   otherwise a future change to the `Default` impl silently loosens
   read-only paths.
3. **Three separate field-to-method renames** (`file_system_sandbox_policy`,
   `network_sandbox_policy`, plus the new `permission_profile`
   accessor implied by the helper) all live as plain methods on
   `Permissions`. The pattern of `let configured_X = self.config.permissions.X();`
   appearing repeatedly suggests these should perhaps return
   borrowed references where possible to avoid clone churn, or at
   least be marked `#[inline]` if they're trivially derived. Visible
   in the `codex_message_processor.rs` change is a fresh `let
   configured_file_system_sandbox_policy = ...` binding before the
   call to `preserve_configured_deny_read_restrictions` — that's a
   clone where a borrow would do.
4. **`from_runtime_permissions_with_enforcement`** is now called
   from app-server with the *enforcement* of the supplied profile +
   the *file system policy* derived from it. The naming is fine but
   the function gets called in a hot path and currently takes
   `&FileSystemSandboxPolicy` by reference and a `NetworkSandboxPolicy`
   by value — uniformity across the two would simplify call sites.
5. The Sapling stack annotation in the PR body shows this PR depends
   on #19391 and is depended on by #19393. Reviewers landing the
   stack out-of-order should bisect carefully — the field→method
   renames in this PR will not cherry-pick cleanly onto a tree that
   doesn't have #19391.

## Verdict — merge-after-nits

The PR achieves its stated goal — collapsing two parallel sources of
truth into one canonical profile with derived legacy projections. The
single non-obvious behavior change (MCP elicitation auto-approval
now covers `Managed { Unrestricted, _ }`) is correctly tested but
under-described in the PR body. Worth landing once that's
acknowledged, ideally with a one-line callout in the PR body or a
regression test pinning `PermissionProfile::default()` ≡ "read-only"
semantics.

## What I learned

Long-running migration stacks like this one tend to leave behind a
"compat surface" that becomes its own attack surface: the moment two
fields can be read independently they will drift, and the cost of
discovering the drift scales with the number of consumers. The
right shape — applied here — is to make the "old" field a *derived
projection* of the "new" canonical state, and force every consumer
to go through a method that has exactly one implementation.
Compounding lesson: when a refactor incidentally widens a
behavioral predicate (here, "auto-approve" now matches a profile
shape the old enum couldn't express), the PR body should call that
out explicitly even if the test matrix already covers it — reviewers
read titles and body more carefully than they read every test
matrix.
