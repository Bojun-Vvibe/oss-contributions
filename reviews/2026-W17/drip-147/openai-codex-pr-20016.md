# openai/codex #20016 — core tests: send model turns with permission profiles

- PR: https://github.com/openai/codex/pull/20016
- Head SHA: `bb39f8104055ea91b2e4799064692b28dc5681af`
- Author: bolinfest (Michael Bolin)
- Files: `codex-rs/core/tests/suite/remote_models.rs` (+14/−7), `codex-rs/core/tests/suite/responses_api_proxy_headers.rs` (+8/−9)
- Stack position: top of a 7-PR Sapling stack (#19900 → #20008 → #20010 → #20011 → #20013 → #20015 → **#20016**)

## Observations

- This is the cap of a multi-PR test-migration stack converting `codex-core/tests` from legacy `SandboxPolicy::DangerFullAccess` direct construction to `PermissionProfile::Disabled` (and `PermissionProfile::workspace_write()`) via the canonical `turn_permission_fields()` helper. Two files left to migrate after #20015 landed; this PR closes them out.
- File 1: `remote_models.rs` (+14/−7) — migrates `Op::UserTurn` construction sites that were inlining `SandboxPolicy::DangerFullAccess` to instead build the permission fields from `PermissionProfile::Disabled` via `turn_permission_fields()`. Net additions are the helper-derived `cwd`/`approval_policy`/`sandbox_policy`/`model`/`effort`/`summary` fields replacing single `sandbox_policy: SandboxPolicy::DangerFullAccess`. Mechanical but correct.
- File 2: `responses_api_proxy_headers.rs` (+8/−9) — migrates the proxy-header helper from inlining a workspace-write `SandboxPolicy` to `PermissionProfile::workspace_write()`. Net negative line count makes sense: builder-pattern helper is more compact than the inline struct construction it replaces.
- Stated goal: "Reduce `SandboxPolicy` references in `codex-rs/core/tests` from 22 files after #20015 to 20 files." This is a measurable invariant that's easy to verify post-merge with `rg 'SandboxPolicy::' codex-rs/core/tests | cut -d: -f1 | sort -u | wc -l`.
- CI scope: `cargo check -p codex-core --tests` + `just fmt`. No behavioral change — only test-construction shape changes — so `cargo check` is a sufficient gate.

## Risks / nits

- `PermissionProfile::Disabled` is a tagged-union variant whose `turn_permission_fields()` projection had better produce *exactly* the field set that `SandboxPolicy::DangerFullAccess` would have produced via the previous direct construction, otherwise tests are silently exercising a different policy under the same name. The earlier PRs in the stack (#20011 "build user turns from permission profiles") presumably added the equivalence test, but worth re-verifying that `Disabled` round-trips to `DangerFullAccess` at the policy-projection layer (not just at the tagged-union layer).
- Two-file PR is the right blast radius for the cap of a stack — but reviewers should NOT treat this as standalone. The behavioral correctness of the migration lives in earlier PRs in the stack (#20011 and #20015 specifically). This PR is a mechanical follow-on.
- No new tests added — appropriate, since this is a test-code refactor, not a feature change. The existing test cases in both files continue to assert their original behavior; only the construction shape moves.
- Post-merge audit: confirm with `rg 'SandboxPolicy::' codex-rs/core/tests` that the count actually drops to 20 files. If it doesn't, there's a stale construction site the migration missed.
- Stack ordering note: this PR depends on #20015 having landed first (which itself depends on #20013). If reviewer is reviewing the stack out of order, behavior of this diff alone is hard to verify.

## Verdict: `merge-as-is`

**Rationale:** Two-file mechanical migration capping a clean Sapling stack, gated by `cargo check -p codex-core --tests` and `just fmt`. The migration shape (`PermissionProfile::Disabled` + `turn_permission_fields()` replacing inline `SandboxPolicy::DangerFullAccess`; `PermissionProfile::workspace_write()` replacing inline workspace-write struct) is consistent with the stack's earlier PRs and the count-of-`SandboxPolicy`-references invariant gives a checkable post-merge gate. Trust the stack's earlier behavioral-equivalence proofs and land this as the natural cap. The only thing I'd actually want to verify post-merge is the `rg | wc -l` count claim.
