---
pr: https://github.com/openai/codex/pull/20733
head_sha: 15bb7f5530243296b72241a4e71e7b79a74d2959
author: abhinav-oai
additions: 1632
deletions: 789
files_touched: 19
---

# Scope

Large refactor that promotes `ApprovalRequest` (renamed from
`GuardianApprovalRequest`) into the canonical in-core description of any
approval-worthy action — shell, unified exec, apply_patch, network, MCP
tool calls, and `request_permissions`. Each producer now builds one
request object that is then projected into:

1. the hook payload (`permission_request_payload()`),
2. the guardian review payload,
3. the human prompt event (existing transport stays stable),
4. the MCP-specific compatibility answer/question-id shaping,

instead of every consumer re-deriving its own shape. MCP prompt details
move from `mcp_tool_call.rs` next to the canonical request in
`guardian/approval_request.rs`. Touches `core/src/codex_delegate.rs`,
`core/src/guardian/*`, `core/src/mcp_tool_call*`, `core/src/session/mod.rs`,
all four tool runtimes (`shell`, `unified_exec`, `apply_patch`,
`network_approval`), plus tests. Net +843 lines but `mcp_tool_call.rs`
shrinks by 248.

# Specific findings

- `codex-rs/core/src/codex_delegate.rs:455` (head `15bb7f5`) — `handle_exec_approval`
  now constructs the `ApprovalRequest::Shell` once and clones into both the
  guardian path and the new
  `parent_session.request_command_approval_for_request(...)` call. Good:
  eliminates the prior duplication where the guardian arm built a
  `GuardianApprovalRequest::Shell` and the user arm passed a parallel arg
  list. Note the new `hook_command: shlex_join(&command)` field is computed
  unconditionally even when the request never reaches the hooks layer —
  cheap but should be lazy if a hot path emerges (not blocking).
- `codex-rs/core/src/codex_delegate.rs:531` — `handle_patch_approval`
  pre-computes the unified `patch` string before deciding whether the
  guardian path is taken. Previously this was only built inside the
  guardian arm. Net result: every patch approval pays the
  serialization cost even when guardian is disabled. For large patches
  (`FileChange::Update` with big diffs) this is a non-trivial allocation
  on the hot path. Suggest gating behind a `OnceCell`/closure or
  computing inside the branch that needs it.
- `codex-rs/core/src/codex_delegate.rs:706` — replaces ~30 lines of
  inline `selected_label` mapping with
  `approval_request.mcp_tool_approval_compat_response(question, decision)`.
  Big readability win and removes the silent assumption that the question
  always carries `MCP_TOOL_APPROVAL_ACCEPT_FOR_SESSION` in its options
  list. Confirm the new helper preserves the historical behavior where a
  missing `ApprovedForSession` option falls back to plain `Accept` — the
  new test
  `delegated_mcp_guardian_abort_returns_synthetic_decline_answer`
  covers the abort case but the fallback case isn't visible in the diff
  excerpt.
- `codex-rs/core/src/guardian/approval_request.rs:1-22` (new imports
  list) — the file now pulls in `RequestUserInputAnswer`,
  `RequestUserInputQuestion`, `RequestUserInputQuestionOption`, and
  `RequestUserInputResponse`. This makes `approval_request.rs` aware of
  the user-input transport, which is fine for now but mildly weakens the
  module's "pure data" character claimed in the PR description ("Kept the
  model focused on what action is being reviewed now"). Worth a follow-up
  to extract the compat-response surface into a sibling module if it
  grows further.
- `codex-rs/core/src/codex_delegate_tests.rs:387` — new test
  `handle_patch_approval_uses_tool_call_id_for_round_trip` asserts the
  `call_id` is reused for both the emitted
  `ApplyPatchApprovalRequest` event and the `Op::PatchApproval` reply.
  Good regression cover for the canonical-request refactor.
- The PR description ("Halt — Maybe de-prio?" is on a *different* PR;
  this one's body is "Summary / Flow / Design decisions / Validation"
  with `cargo test -p codex-core` invocations enumerated). Validation
  coverage looks proportionate to the blast radius; fmt + `just fix` ran.

# Suggested changes

1. Defer the `patch` string construction in `handle_patch_approval` until
   the guardian arm actually needs it (or compute lazily) to avoid
   regressing memory pressure on big diffs in non-guardian configs.
2. Confirm `mcp_tool_approval_compat_response` has explicit test coverage
   for the case where `question.options` does *not* contain
   `MCP_TOOL_APPROVAL_ACCEPT_FOR_SESSION` and the decision is
   `ApprovedForSession` — the original code fell back to
   `MCP_TOOL_APPROVAL_ACCEPT`; preserving that behavior should be pinned
   by a unit test in `approval_request::tests`.
3. Consider a follow-up that splits `approval_request.rs` into a pure
   data module + a `compat::user_input` adapter module to keep the
   transport-agnostic claim honest as more compatibility surfaces are
   added.

# Verdict

`merge-after-nits`

# Confidence

Medium-high. The refactor is well-motivated and the test list is
substantive; flagged items are localized and easy to address either in
this PR or as fast follow-ups.

# Time spent

~14 minutes.
