# openai/codex PR #19395 — permissions: finish profile-backed app surfaces

- **Author:** bolinfest
- **Head SHA:** 2f73fcaa20e33624d5d514e70744867de061ea28
- **Files:** `codex-rs/analytics/src/reducer.rs` (+10 / −7),
  `codex-rs/app-server/src/codex_message_processor.rs` (+43 / −19),
  `codex-rs/app-server/src/lib.rs` (+1 / −1),
  `codex-rs/exec/src/event_processor_with_human_output.rs` (+69 / −40),
  `codex-rs/exec/src/event_processor_with_human_output_tests.rs` (+72),
  `codex-rs/exec/src/lib.rs` (+7 / −51),
  `codex-rs/sandboxing/src/bwrap.rs` (+10 / −7),
  `codex-rs/sandboxing/src/lib.rs` (+1 / −1),
  `codex-rs/tui/src/app/startup_prompts.rs` (+3 / −3)
- **Verdict:** `merge-after-nits`

## What the diff does

Final cleanup pass in the long-running migration off
`SandboxPolicy::to_legacy_sandbox_policy(cwd)` toward
`PermissionProfile::file_system_sandbox_policy()` + its
`has_full_disk_write_access` / `get_writable_roots_with_cwd` /
`can_write_path_with_cwd` query methods. Each touched site previously
did `match profile.to_legacy_sandbox_policy(cwd)` → enum branches;
this PR replaces those with direct profile queries.

Concrete changes:
- `analytics/src/reducer.rs` lines 962–977: `sandbox_policy_mode`
  no longer round-trips through `SandboxPolicy::DangerFullAccess /
  ReadOnly / WorkspaceWrite`. Instead it asks
  `file_system_policy.has_full_disk_write_access()` and
  `get_writable_roots_with_cwd(cwd).is_empty()`.
- `app-server/src/codex_message_processor.rs` lines 2596–2608:
  collapses the old `requested_permissions_trust_project ||
  matches!(... WorkspaceWrite | DangerFullAccess |
  ExternalSandbox)` chain into a single
  `effective_permissions_trust_project` derived from the new
  `permission_profile_trusts_project` helper at lines 9967–9979.
- `requested_permissions_trust_project` (line 9955–9963) becomes a
  one-line delegate to `permission_profile_trusts_project`,
  eliminating the duplicated enum-match branches.
- `permission_profile_trusts_project` (new, lines 9967–9979): the
  central definition — `Disabled` and `External { .. }` are
  unconditionally trusted; `Managed { .. }` is trusted iff
  `file_system_sandbox_policy().can_write_path_with_cwd(cwd, cwd)`.
- `exec/src/lib.rs` shrinks by ~44 lines net (51 deletions, 7
  additions): the legacy mapping helper is gone.
- `exec/src/event_processor_with_human_output.rs` (+69 / −40) and
  its test file (+72 new lines) reflect the same shift to
  profile-driven querying for the human-readable status banner.
- `sandboxing/src/bwrap.rs` (+10 / −7) and `sandboxing/src/lib.rs`
  (+1 / −1): touch points where bwrap arg construction asked the
  policy what writable roots to bind-mount.
- `tui/src/app/startup_prompts.rs` (+3 / −3): startup banner copy
  reflecting the new query API.

The new test added at `codex_message_processor.rs` lines 10510–10524
exercises the "split write profile" case — a `Managed` profile that
allows write to `cwd` but denies a glob pattern `/tmp/project/**/*.env`.
That's the case the legacy mapping couldn't represent and is the
whole point of the migration.

## Review notes

- **Behavior parity is the live question.** The old code
  trusted any policy that mapped to
  `WorkspaceWrite | DangerFullAccess | ExternalSandbox`. The new
  code trusts `External { .. }` and `Disabled` unconditionally, and
  `Managed { .. }` iff it can write to `cwd`. That's *almost* the
  same set, but a `Managed` profile that grants write to *some
  other path* but not `cwd` would have mapped to `WorkspaceWrite`
  under the old code (and trusted) but is now untrusted. That seems
  like the *correct* tightening — you shouldn't auto-trust a
  project just because the user has write somewhere unrelated — but
  it should be called out explicitly in the PR description as an
  intentional behavior change, not a refactor.
- **`Disabled` → trust is a strong claim.** `Disabled` means "no
  sandbox at all", so trusting the project is consistent (there's
  nothing to gate against), but a comment in
  `permission_profile_trusts_project` explaining *why* the
  no-sandbox case is treated as trusted-project would help future
  readers. Currently it reads like an afterthought.
- **Test coverage for the new helper** is asserted at
  `codex_message_processor.rs:10507–10524` for the split-write
  managed case; worth adding a parallel test for the
  `Disabled` and `External` branches so the trust-decision matrix
  is fully pinned down. If `Disabled` semantics change in the
  future, the test failure should pinpoint the regression.
- **`exec/src/lib.rs` shrinks by 44 lines** because the legacy
  `to_legacy_sandbox_policy` consumer is gone. Worth a quick
  `rg "to_legacy_sandbox_policy" codex-rs` to confirm no callers
  remain — if the migration is complete, the method itself can
  be marked `#[deprecated]` or removed in a follow-up, which would
  prevent future code from accidentally re-introducing the round
  trip.
- **`bwrap.rs` change** turns the mapping into a direct
  `file_system_policy` query. That's the correct shape — bwrap's
  `--bind` / `--ro-bind` arg construction is a pure derivation of
  writable roots — but the old code may have had subtle ordering
  guarantees (system roots before user roots, etc.) that aren't
  obvious from the diff. Worth confirming the bwrap arg list is
  byte-identical for the common `WorkspaceWrite` profile via a
  golden-file test.

Final-stage migrations like this are usually mostly mechanical, and
this one is — the value is in the small set of *intentional*
behavior tightenings buried in the refactor, which the PR
description should make explicit before merge.

## What I learned

When a "legacy adapter" function (`to_legacy_sandbox_policy`) sits
between a new richer model and the rest of the codebase, finishing
its removal is mostly mechanical *except* at the leaf sites where
the adapter quietly papered over expressiveness gaps. Splitting
those out (here: the split-write profile case) is the only test
that proves the new model is genuinely more capable, not just
isomorphic.
