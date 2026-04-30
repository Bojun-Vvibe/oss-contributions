# Review: openai/codex#20404 — rollout tests: seed turn contexts from permission profiles

- PR: https://github.com/openai/codex/pull/20404
- Author: bolinfest (Michael Bolin)
- headRefOid: `f5993049763d12bdf3cbec6808725dce424c4ded`
- Files: `codex-rs/rollout/src/recorder_tests.rs` (+4/-3)
- Verdict: **merge-as-is**

## Analysis

This is a tiny, surgical change inside the `resume_candidate_matches_cwd_reads_latest_turn_context`
recorder test (around `recorder_tests.rs:1095-1112`). It removes the last test-only consumer of
`SandboxPolicy::new_read_only_policy()` from the rollout crate and instead seeds the
`TurnContextItem` fixture from `PermissionProfile::read_only()`, then derives the legacy
`sandbox_policy` field via `to_legacy_sandbox_policy(latest_cwd.as_path())` and populates
`permission_profile: Some(permission_profile)`. That makes the fixture shape match what the
runtime now actually writes into rollout records — the `permission_profile` field is no longer
forced to `None`, which more accurately exercises the resume-candidate matcher.

The diff also drops the `use codex_protocol::protocol::SandboxPolicy;` import in favor of
`use codex_protocol::models::PermissionProfile;`. That's the correct trade — once the legacy
constructor is no longer called, the import is dead. No production code is touched, so blast
radius is restricted to one async test.

The one thing I'd want to verify is that `PermissionProfile::read_only().to_legacy_sandbox_policy(cwd)`
yields a value byte-equal to the previous `SandboxPolicy::new_read_only_policy()` for purposes of
serialization in the JSONL rollout file — since this test reads the latest turn context back through
`resume_candidate_matches_cwd`, any drift in the legacy projection would silently change the resume
matching surface. Spot-checking the protocol crate's `to_legacy_sandbox_policy` should confirm; the
PR body says `cargo test -p codex-rollout resume_candidate_matches_cwd_reads_latest_turn_context`
passes, which is the right signal.

Part of a long Sapling stack (PRs #20355 through #20404) migrating the entire codebase off the
cwd-less legacy bridge. As a leaf in that stack, this is correctly minimal — merge as-is.
