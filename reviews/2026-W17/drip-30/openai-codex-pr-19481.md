# openai/codex#19481 — Remove ghost snapshots from the Responses API

**Verdict: merge-after-nits**

- PR: https://github.com/openai/codex/pull/19481
- Author: pakrym-oai
- Base SHA: `8a559e7938bd841d78dc2c442deaa76424faf1d9`
- Head SHA: `439f4944d3a6db304cd2fb57ca093f3989736216`
- +1206 / -3639

## Summary

Hard removes the `ghost_snapshot` / `GhostCommit` feature: deletes
`codex-rs/git-utils/src/ghost_commits.rs` (-1786 lines), the
`tasks/ghost_snapshot.rs` task module (-252), and the entire ghost-undo
test suite at `core/tests/suite/undo.rs` (-516). Schemas (`ClientRequest.json`,
`codex_app_server_protocol.{,v2.}schemas.json`, `v2/RawResponseItem...`,
`v2/ThreadResumeParams.json`) drop the `GhostCommit` definition and the
`GhostSnapshotResponseItem` variant. Undo becomes a no-op surfacing
"feature is unavailable"; legacy config keys are still accepted to avoid
breaking on-disk profiles.

## Specific findings

- `codex-rs/protocol/src/models.rs:+20/-63` — the `ResponseItem` enum loses
  the `GhostSnapshot { ghost_commit }` variant. This is a wire-format
  break for any rollout file already containing a ghost-snapshot item.
  `rollout/src/policy.rs:-2` confirms the policy entry is removed too,
  so older rollouts will fail to deserialize with `unknown variant
  ghost_snapshot`. There's no migration shim or `#[serde(other)]` arm —
  needs at least a forward-compat skip-and-warn or a one-version
  deprecation cycle.
- `codex-rs/core/src/config/mod.rs:+23/-2` — keeps reading the legacy
  `experimental_ghost_snapshots` (and similar) keys. Good: prevents
  startup failures on existing user configs. Worth a one-line warn log
  the first time a removed key is observed so users know to delete it.
- `codex-rs/core/src/tasks/undo.rs:+2/-59` — `UndoTask::run` collapses to
  emit a "feature unavailable" message. The string surfaces in the TUI
  popup snapshot diff at
  `tui/src/chatwidget/snapshots/codex_tui__chatwidget__tests__experimental_features_popup.snap:+4/-2`.
  Reasonable for a deletion PR, but the snapshot churn means any
  follow-up to the popup will conflict — fine.
- `sdk/python/src/codex_app_server/generated/v2_all.py:+1062/-381` — net
  +681 lines in a *deletion* PR is suspicious; it's regenerator drift
  (formatting + reordered enum members). Worth a maintainer eyeball to
  confirm the SDK is byte-identical to a fresh `regen` from a clean
  checkout, not a stale local generation.
- `codex-rs/features/src/tests.rs:+22` — adds tests asserting the
  removed feature flag is no longer registered. Good defensive coverage
  to prevent revival via accidental re-add.

## Rationale

The deletion is mechanically clean and the protocol direction is
correct — ghost snapshots were experimental, never stabilized, and the
parallel `git-utils/ghost_commits.rs` was a 1786-line maintenance
liability paired with a 516-line test suite that locked in semantics
nobody is committed to. The legacy-config-load carve-out is the right
call to avoid hard-breaking users mid-flight, and the no-op undo
message is an honest signal rather than silent acceptance. My one
real concern is the rollout/wire-format break: `ResponseItem` drops a
variant that has shipped to disk, and there's no skip-unknown or
serde `other` arm in the deserialize path I can see. A user resuming
an old session that contains a ghost-snapshot item will see a
deserialization error and lose the resume. A `#[serde(other)]` catch-all
or a one-version `#[deprecated]` keep-and-ignore on the `GhostSnapshot`
variant would cleanly cover that. The +1062-line SDK churn also wants
a sanity check that it's pure regen — easy to confirm by running the
generator on `main` and diffing. The schema-removal arms (5 JSON files,
all -52 lines each) line up exactly, which is a good sign the schema
generator was rerun consistently.
