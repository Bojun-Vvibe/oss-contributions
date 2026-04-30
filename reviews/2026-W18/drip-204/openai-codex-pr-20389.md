# openai/codex PR #20389 — tests: use disabled profile in exec capture check

- **URL:** https://github.com/openai/codex/pull/20389
- **Head SHA:** `2876493bae2035b918fee37e29038b576fe8b45f`
- **Diff size:** +1 / -2 in `codex-rs/core/src/exec_tests.rs`
- **Verdict:** `merge-as-is`

## Specific diff references

- `codex-rs/core/src/exec_tests.rs:347-351` — the only change. The previous two lines constructed `SandboxPolicy::DangerFullAccess` and immediately funneled it through `PermissionProfile::from_legacy_sandbox_policy(&sandbox_policy)`. The new single line is `let permission_profile = PermissionProfile::Disabled;`, which is the direct enum variant the legacy bridge would have produced.
- The function under test (`process_exec_tool_call_preserves_full_buffer_capture_policy`) is unchanged in its assertions; it still passes `permission_profile` into `process_exec_tool_call` via the same `ExecParams { command, ... }` struct literal at the call site immediately below the diff.
- Stack context from the PR body lists this as the leaf of a 25-PR Sapling stack moving the broader codebase from `SandboxPolicy` legacy plumbing onto `PermissionProfile` directly. PRs #20388, #20387, #20386, #20384, etc. apply the identical mechanical swap in adjacent test files, so this PR's risk surface is a single function in a single test module.

## Reasoning

This is the cleanest possible test refactor: net -1 line, zero behavior change, removes a dependency on a legacy abstraction (`SandboxPolicy::DangerFullAccess` → `PermissionProfile::from_legacy_sandbox_policy`) that the rest of the stack is in the process of retiring. The variant `PermissionProfile::Disabled` is exactly what `from_legacy_sandbox_policy(&SandboxPolicy::DangerFullAccess)` returns by construction, so the test's effective input is identical and the assertion surface is untouched.

The contributor explicitly notes that other `SandboxPolicy` references in the same file are intentionally left in place because they exercise Windows sandbox compatibility paths that still consume legacy policy input — that's the correct scoping decision, since flipping those would actually change what's being tested. The `just fix -p codex-core` and targeted `cargo test` invocation listed in the PR body are sufficient verification for a one-line test change of this shape.

CI risk is nil; reviewer time is well-spent on the bigger PRs in the stack rather than this leaf. Merge as-is and let the Sapling stack land. The only thing I'd suggest the stack lead consider — outside this PR's scope — is squashing the 25 leaf PRs into a smaller number of logical groups before final merge to keep `git log` readable.
