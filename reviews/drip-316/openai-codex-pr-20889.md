# Review: openai/codex #20889 — Plan Windows protected metadata targets

- PR: https://github.com/openai/codex/pull/20889
- Head SHA: `1b1c053af60630f07486c492e6649157c6831f32`
- Author: evawong-oai
- Size: +210 / -1

## Summary
First in a 3-PR stack (this → #20890 → #20891) that adds Windows
sandbox enforcement for "protected metadata" paths like `.git`,
`.codex`, `.agents`. This PR introduces the planning types
(`WindowsProtectedMetadataTarget`, `WindowsProtectedMetadataMode`) and
the `windows_protected_metadata_targets()` resolver that walks
`WritableRoot::protected_metadata_names`, normalizes each path, and
classifies it as `ExistingDeny` (path exists → ACL deny-write) or
`MissingCreationMonitor` (path absent → install a creation monitor so
attacker-controlled creation can be detected/blocked).

## Specific citations
- `codex-rs/core/src/exec.rs:111` — adds
  `protected_metadata_targets: Vec<WindowsProtectedMetadataTarget>` to
  `WindowsSandboxFilesystemOverrides`. This is the data carrier
  consumed downstream by #20890.
- `codex-rs/core/src/exec.rs:115-129` — `Ord`/`PartialOrd` derived on
  the target struct so the `BTreeSet` collection at :1310 produces a
  deterministic, dedup'd order. Good — sandbox config should be
  reproducible across runs.
- `codex-rs/core/src/exec.rs:1152-1156` — short-circuit guard now
  checks both `additional_deny_write_paths.is_empty()` AND
  `protected_metadata_targets.is_empty()` before returning `None`.
  Correct — if either has entries we must build overrides.
- `codex-rs/core/src/exec.rs:1287-1295` — same short-circuit for the
  elevated path. Symmetric with the restricted-token path.
- `codex-rs/core/src/exec.rs:1308-1325` — `windows_protected_metadata_
  targets`. The `BTreeSet` insertion order means duplicates across
  multiple writable roots collapse to one entry. The mode is
  determined per-path via `windows_protected_metadata_mode()` which
  uses `symlink_metadata` (not `metadata`) — important so a symlink
  pointing nowhere is still classified as "existing" rather than
  silently treated as missing. Good.
- `codex-rs/core/src/exec.rs:1326-1332` — `MissingCreationMonitor` is
  the fallback when `symlink_metadata` returns `Err`. This conflates
  "ENOENT" with "permission denied" — on Windows that probably matters
  little since the sandbox runs in the same security context as the
  parent, but worth documenting.
- `codex-rs/core/src/exec_tests.rs:666-680` and `:778-792` — both the
  restricted-token and elevated paths now expect three default
  protected metadata targets (`.agents`, `.codex`, `.git`) each in
  `MissingCreationMonitor` mode. Tests pass without creating those
  dirs, so the default `WritableRoot::protected_metadata_names` is
  emitting them unconditionally.
- `codex-rs/core/src/exec_tests.rs:725-770` — new test
  `windows_metadata_plan_marks_existing_metadata_for_deny`
  pre-creates `.git` then asserts the resolver flips that one entry
  from `MissingCreationMonitor` to `ExistingDeny`. Direct coverage of
  the classification branch — nice.

## Verdict
**merge-after-nits**

## Nits
1. `windows_protected_metadata_mode()` swallows all `symlink_metadata`
   errors as "missing". Add a `tracing::trace!` for the non-ENOENT
   case so future debugging of "why was this path treated as missing"
   has a breadcrumb.
2. The PR title says "Plan" but the code actually adds the planning
   types AND the resolver. Consider renaming to "Plan Windows
   protected metadata targets (resolver only, no enforcement)" so
   reviewers know the boundary with #20890/#20891.
3. The default set (`.agents`, `.codex`, `.git`) baked into
   `WritableRoot::protected_metadata_names` isn't shown in this diff
   — link the defining PR/file in the description.
