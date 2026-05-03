# Review: openai/codex PR #20733

- **Title:** centralize approval prompts
- **Author:** abhinav-oai (Abhinav)
- **Head SHA:** `dbff525377e7e41f092493604e537cf3550fe0fc`
- **Verdict:** needs-discussion

## Summary

Refactor: collapses four parallel "describe an action requesting
approval" code paths down to a single canonical
`GuardianApprovalRequest` enum. Producers (shell, unified exec,
execve, apply_patch, network, MCP, request_permissions) build one
request object, and downstream consumers project from it: hook
payloads via `permission_request_payload()`, guardian review by
serializing the request directly, the user-facing transport event /
MCP prompt by re-projecting from the same request, and delegated
sub-agent approvals by reusing the request instead of reconstructing
in the bridge. 1298 added / 501 deleted across 16 files in
`codex-rs/core`.

## Specific-line comments

- `codex-rs/core/src/guardian/approval_request.rs:+419` (new file) —
  this is the central type. Variants visible from the diff include
  `Shell { id, command, hook_command, cwd, sandbox_permissions,
  additional_permissions, justification }` and `ApplyPatch { id, cwd,
  files, changes, patch }`. Shell carries both the structured
  `command` (Vec<String>) AND the `hook_command` (joined
  `shlex_join`'d string) — verify that's intentional rather than
  having `hook_command` be a derived method on the struct. Storing
  both is denormalized state.
- `codex-rs/core/src/codex_delegate.rs:452-490` — the new shape: build
  one `approval_request` up front, `clone()` it for the guardian
  branch, pass the original to
  `request_command_approval_for_request`. Clone-per-branch is fine
  for a struct of this size; not a hot path.
- `codex-rs/core/src/codex_delegate.rs:551-580` — patch approval
  branch builds the `patch` string (the `*** Add File: ...` /
  `*** Update File: ...` synthesis) at the top of the function
  unconditionally even when `routes_approval_to_guardian` is false,
  because both branches now consume the same
  `GuardianApprovalRequest::ApplyPatch { ..., patch }`. Previously
  this string was only built inside the guardian branch. **This is a
  behavior change in the non-guardian path: it now does the patch
  serialization work even when the guardian path is off.** For large
  patches this is non-trivial work. Probably acceptable but worth
  benchmarking.
- `codex-rs/core/src/codex_delegate.rs:704-712`
  (`mcp_tool_approval_compat_response`) — extracted helper replaces
  the inline match on `ReviewDecision` that previously hardcoded
  `MCP_TOOL_APPROVAL_ACCEPT` / `..._FOR_SESSION` /
  `..._DECLINE_SYNTHETIC` label lookups. Centralizing the
  ReviewDecision → MCP-label projection is the right call. Verify
  the helper's signature accepts the question's options and
  preserves the "find by label" behavior so existing MCP servers
  with custom approval-option labels still work.
- `codex-rs/core/src/mcp_tool_call.rs:+194/-106` — the bulk of the
  refactor lands here. Without the full diff visible it's hard to
  audit, but the test file
  `codex-rs/core/src/mcp_tool_call_tests.rs` grew +273 / -99, which
  is a reassuring sign that the new code paths got dedicated test
  coverage. Reviewer should walk that test diff carefully.
- `codex-rs/core/src/session/mod.rs:+90/-75` — net +15 in the session
  module is consistent with extracting the request build out of
  per-method argument lists into a `*_for_request` family. The PR
  description claims existing protocol stability — confirm that
  `ExecApprovalRequestEvent` etc. on the wire are unchanged.

## Risks / nits

- This is a large refactor (1298+/501-) across the core approval
  surface. Even if every line is correct, the **review surface** is
  the issue: 16 files, 8 of them in `tools/runtimes/`, all changing
  signatures from positional args to `*_for_request(request)`. A
  pre-merge checklist should confirm every existing caller migrated
  and no non-`_for_request` legacy method survives unintentionally.
- No mention of behavior preservation for the **denied** /
  **timed out** / **abort** ReviewDecision paths in the PR
  description. The new `mcp_tool_approval_compat_response` collapses
  three of those into one synthetic decline label. Confirm this
  matches prior behavior for each — a regression here would silently
  change what gets reported to the MCP server.
- The `clone()` of `GuardianApprovalRequest` at every call site is
  harmless for shell/patch but `ApplyPatch` carries `changes:
  HashMap<PathBuf, FileChange>` and `patch: String`, both of which
  can be large. For 100MB patches this is 200MB+ of clones.
  Acceptable (those don't happen in practice) but document it.

## Verdict justification

The refactor is well-motivated and the producer/consumer separation
is the right abstraction. But: (1) the unconditional patch
serialization in the non-guardian branch is a real (small) perf
behavior change, (2) the MCP decline-path collapse needs explicit
verification, and (3) the review surface is large enough that a
maintainer should walk through every `_for_request` migration site
before approving. Worth a discussion thread on the PR before merge,
not a flat reject. **needs-discussion.**
