# openai/codex#20446 — protocol: drop legacy sandbox from user turns

- PR: https://github.com/openai/codex/pull/20446
- Head SHA: `dd07fb5a2a94cd2c22281a3ccefe2d585d4a4d5c`
- Author: bolinfest
- Files: 42 changed, +137/−265 (net −128). Sapling stack, parent of #20441.

## Context

Two PRs ago in the same stack (#20441), `Op::UserTurn` got `permission_profile`
promoted from `Option<...>` to required. That made the
`sandbox_policy: Option<SandboxPolicy>` field on the same op a dead slot —
core never read it once a profile was present, but every callsite still
threaded a `None` through it and every test had to construct one. This PR
deletes the field.

## Design

The protocol-level deletion is one line:
`codex-rs/protocol/src/protocol.rs` loses 9 lines (the `sandbox_policy: ...`
field on `Op::UserTurn`). `codex-rs/tui/src/app_command.rs` loses 3
correspondingly.

The producer side gets two destructure-pattern updates:
- `core/src/session/handlers.rs:134` — `sandbox_policy: _` is removed from
  the destructure of `Op::UserTurn`, so the binding doesn't shadow.
- `core/src/guardian/review_session.rs:713` and
  `core/src/session/tests.rs:4247` — `sandbox_policy: None,` lines are
  deleted from the two remaining direct constructors.

The bulk of the diff (37 of 42 files) is mechanical test updates renaming the
helper from `turn_permission_fields` to `turn_permission_profile` in
`core/tests/common/test_codex.rs:202-218`:

```rust
pub fn turn_permission_profile(
    permission_profile: PermissionProfile,
    _cwd: &Path,
) -> PermissionProfile {
    permission_profile
}
```

The old helper returned a tuple `(Option<SandboxPolicy>, PermissionProfile)`
where the first element was always `None` after the previous PR — see the
deleted comment "The legacy sandbox field is intentionally omitted when a
canonical PermissionProfile is available." The rename + return-type collapse
forces every callsite to stop pretending it's threading a sandbox policy.

Each of the 35 test files in `core/tests/suite/*.rs` gets an identical
two-edit pattern:
```rust
- let (sandbox_policy, permission_profile) =
-     turn_permission_fields(permission_profile, ...);
+ let permission_profile = turn_permission_profile(permission_profile, ...);
```
plus the corresponding `sandbox_policy,` line removed from the
`Op::UserTurn { ... }` literal. Spot-checked at
`apply_patch_cli.rs:64-78`, `approvals.rs:590-605`, `client.rs:1726-1755`,
`code_mode.rs:2610-2625`, `compact.rs:73-91`, and the 9-deletion stretch in
`model_visible_layout.rs` — all identical pattern, no semantic drift.

## What's deliberately *not* removed

The PR description calls it out: `UserInputWithTurnContext` and
`OverrideTurnContext` keep `sandbox_policy` because older
serialized/replayed payloads still carry it. That's the right scope —
`Op::UserTurn` is the in-process API surface, the other two are the
on-disk/wire compat surface, and they have different deprecation cadences.

## Risks / nits

- 42-file PR with 35 mechanical test edits is hard to review by eyeball;
  the right review move is `git log -p --grep='sandbox_policy' -- 'codex-rs/core/tests/suite/*.rs'`
  on the PR branch and confirm every diff is exactly the two-edit pattern.
  I sampled 6/35 and they were all clean.
- `_cwd: &Path` on the new helper is now genuinely unused (the prefix `_`
  acknowledges this). Worth a follow-up to drop the parameter entirely once
  no callers need a cwd-dependent variant — currently it's dead weight on
  every callsite.
- The Sapling stack has 60+ ancestors; this PR's review is meaningful only
  in the context of #20441 (which made `permission_profile` required) and
  #20433 (which made the producer-side field optional first). Reviewing
  this PR in isolation looks like a no-op deletion; in stack context it's
  the cleanup that makes the previous two correct.

## Verdict

**`merge-as-is`**

Pure deletion of a now-dead field, mechanical test updates following one
exact pattern, scope correctly limited to `Op::UserTurn` (not the on-disk
compat surfaces). Net −128 lines is the right direction. The verification
list (`cargo check -p codex-protocol -p codex-core -p codex-tui --tests`,
plus targeted `cargo test` runs) is the right gate.

## What I learned

The cost of a "keep the field around for compat" decision compounds: every
new test, every new direct-constructor site has to thread the dead value,
and every reviewer has to ask "do I need to set this?" The discipline of
making `permission_profile` required first (#20441), *then* deleting the
shadow field one PR later (this one), is the right two-step. Doing it as a
single PR would conflate two changes — the contract tightening and the
removal of the now-dead field — and make a revert of either one a partial
revert of both. The Sapling-stack convention of one logically-atomic change
per PR pays off exactly here.
