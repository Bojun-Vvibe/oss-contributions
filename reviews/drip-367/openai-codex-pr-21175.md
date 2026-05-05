# openai/codex #21175 — Wire missing Windows metadata to deny sentinel

- **Head SHA:** `8f93be5b9e21e89802d601fb67596023c858d076`
- **Base:** `codex/windows-protected-metadata-deny-sentinel`
- **Author:** evawong-oai
- **Size:** +33 / −28 across 5 files
  (`codex-rs/cli/src/debug_sandbox.rs`, `codex-rs/core/src/exec.rs`,
  `codex-rs/core/src/exec_tests.rs`,
  `codex-rs/core/src/unified_exec/process_manager.rs`,
  `codex-rs/core/src/unified_exec/process_manager_tests.rs`)
- **Stack position:** PR 20 of 20 in the Windows protected-metadata
  stack
- **Verdict:** `merge-as-is`

## Summary

Final stack PR that flips the policy adapter from
`MissingCreationMonitor` to `MissingDenySentinel` for absent protected
metadata paths (`.agents`, `.codex`, `.git`). After this lands, when
the sandbox sees a *missing* protected metadata target, it creates a
deny-listed sentinel before the command runs instead of just
monitoring for creation, so the sandboxed process is blocked from
materializing a competing copy in the first place.

## What's right

- **Renaming the variant where it lives.** The enum redefinition at
  `codex-rs/core/src/exec.rs:127-135` doesn't just rename it — it
  rewrites the docstring to describe the *new* semantics ("create and
  deny-list a temporary sentinel before command execution can begin")
  and bumps the layer comment to explain the deny-vs-monitor
  distinction at `exec.rs:121-126`. So future readers see the policy,
  not just the symbol.

- **Adapter mapping stays exhaustive.** Both adapter sites
  (`exec.rs:671-675` and
  `unified_exec/process_manager.rs:190-194`) update the
  `WindowsProtectedMetadataMode → ProtectedMetadataMode` match arms in
  lockstep. Because Rust enforces exhaustive matching on these enums,
  the rename surfaces any forgotten call site at compile time —
  there's no risk of an old `MissingCreationMonitor` arm silently
  remaining and routing to a stale sandbox mode.

- **The two top-level decision points are both flipped.**
  `windows_protected_metadata_mode()` at `exec.rs:1366-1369` (the
  policy planner used by core exec) and the equivalent block in
  `cli/src/debug_sandbox.rs:478-484` (the debug-sandbox driver) are
  both updated, so behavior is consistent between the CLI debug path
  and the production exec path. If only one had been flipped, debug
  reproductions would have diverged from real runs — exactly the
  failure mode that makes Windows sandbox bugs hard to chase.

- **Tests are updated, not deleted.** Five existing tests
  (`windows_restricted_token_supports_full_read_split_write_read_carveouts`,
  `windows_elevated_supports_split_write_read_carveouts`,
  `windows_metadata_plan_marks_existing_metadata_for_deny`,
  `windows_shell_runtime_path_resolves_metadata_overrides`,
  `open_session_prepares_windows_metadata_overrides_for_unified_exec`)
  swap their expected mode in place at `exec_tests.rs:669, 781, 843,
  907, 976` and `process_manager_tests.rs:186, 190, 194`. This pins
  the new semantics into the regression suite. The renamed test
  `windows_metadata_plan_uses_sentinel_for_nested_missing_git`
  (`exec_tests.rs:858`) has a name that now matches its body — small
  but appreciated.

- **Test for `ExistingDeny` is preserved.** In
  `windows_metadata_plan_marks_existing_metadata_for_deny`
  (`exec_tests.rs:840-855`), the `.git` arm still expects
  `ExistingDeny` while `.agents` / `.codex` get
  `MissingDenySentinel` — confirming the existing-vs-missing branch
  still fires correctly and this PR only changes the missing branch's
  outgoing mode.

## Risk

- **Single-stack risk.** Stated as the final wire-up of behavior that
  the rest of the 20-PR stack already established. So the runtime
  pieces (`codex_windows_sandbox::ProtectedMetadataMode::MissingDenySentinel`
  itself) need to actually exist in the sandbox crate. Assuming the
  earlier stack PRs landed (they must have, or this wouldn't compile),
  this is mechanically safe.
- **Validation note in PR description: "Stack head local validation
  passed; Windows VM end-to-end validation will run against this final
  stack head."** That's the right gate — this PR is the one that
  actually makes the stack do something user-visible on Windows, so
  E2E on the *final* head is what matters, not on intermediate PRs.
  No concern as long as that E2E result is captured before merge.
- **No behavior change for non-Windows users.** All edits are inside
  Windows-specific modules / branches; Unix sandbox is unaffected.

## Nits

None worth blocking on. The diff is exactly the variant rename,
exhaustive adapter mapping, test expectation updates, and one test
rename. No drive-by changes.

## Verdict

`merge-as-is`, conditional on the Windows VM E2E run on this head
passing. The diff itself is clean, exhaustive, and consistent with the
rest of the stack.
