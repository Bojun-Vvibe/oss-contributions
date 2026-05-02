# openai/codex PR #20737 — [codex] centralize approval routing

- **PR**: https://github.com/openai/codex/pull/20737
- **Head SHA**: `d0d7aacaee35973913346d9b377b91fefdeb06d6`
- **Size**: +853 / −531 across 14 files
- **Verdict**: **needs-discussion**

## What changed

Introduces a new `codex-rs/core/src/tools/approval.rs` module (+526 lines) defining `ApprovalRequest`, `ApprovalRequestKind` (an enum that wraps `McpToolCallApprovalRequest`, presumably plus shell/patch/network variants), `ApprovalOutcome`, `ApprovalDecisionSource` (`PermissionRequestHook | Guardian { review_id } | User`), and a single chokepoint `review_before_user_prompt(sess, ctx, guardian_review_id, evaluate_permission_request_hooks, &approval_request)` that runs hooks → guardian review → user prompt in a fixed order. Callers are migrated to this surface:
- `mcp_tool_call.rs` (lines 1004–1078 of diff): the previous explicit `run_permission_request_hooks(...)` + `routes_approval_to_guardian` + `review_approval_request` two-step is replaced with one `review_before_user_prompt` call, then a `match outcome.source` dispatch translates `PermissionRequestHook` / `Guardian` outcomes into `McpToolApprovalDecision`. The `User` arm is `unreachable!()`.
- `tools/runtimes/apply_patch.rs` (+38 / −58), `shell.rs` (+41 / −49), `unified_exec.rs` (+41 / −53), `network_approval.rs` (+43 / −67), `runtimes/shell/unix_escalation.rs` (+23 / −72): all collapse into the new shared path.
- `tools/sandboxing.rs` shrinks (+4 / −95) — `ApprovalStore` and `PermissionRequestPayload` move to `tools/approval.rs`. Imports in `session/mod.rs:295` and `state/service.rs:13` are rewritten to point at the new location.
- `orchestrator.rs` loses 79 lines of bespoke routing (+22 / −79).
- `session/tests.rs:897` updates the test mock to return the new `ApprovalOutcome { decision, rejection_message, source: ApprovalDecisionSource::User }` instead of a bare `ReviewDecision`.

## Why this matters

Five different runtimes (mcp, shell, apply_patch, unified_exec, network) each had their own copy of "run hooks, then maybe guardian, then maybe prompt user, then translate the answer back into my tool's verdict enum" logic. That's a known source of bugs: a hook decision in one runtime could get applied while another runtime applied guardian-first ordering, and per-runtime decision translation made it easy for one path to honor `ApprovedForSession` caching while another silently dropped it. Centralizing into `ApprovalRequest` + `review_before_user_prompt` is exactly the right structural fix.

That said, this PR is **large and load-bearing for security-critical control flow**. It's the central decision point for every tool that can mutate the user's filesystem, run shell, or open network. It deserves more than a 14-minute reading.

## Specific call-outs

- `mcp_tool_call.rs` line ~1078: `crate::tools::approval::ApprovalDecisionSource::User => unreachable!("review_before_user_prompt never returns user-prompt outcomes")`. Making this `unreachable!()` rather than a graceful fallback is fragile — if the hook path or guardian path ever needs to delegate to the user (e.g. on a future "ask" verdict), this will panic in production. Prefer `debug_assert!` + a defensive `McpToolApprovalDecision::Decline { … }` so worst case is denied-by-default rather than process abort.
- The `evaluate_permission_request_hooks: bool` parameter on `review_before_user_prompt` is a code-smell flag-argument; some callers will pass `true`, some `false`, and from the diff alone it's not obvious *which* runtimes opt out and why. Document this contract in the module doc-comment with an enumeration of which call sites pass which value.
- The move of `ApprovalStore` from `tools/sandboxing.rs` to `tools/approval.rs` is reasonable, but breaks a dependency invariant if any external crate imports `codex_core::tools::sandboxing::ApprovalStore`. A `pub use` re-export at the old path would smooth this for anyone tracking main.
- Test coverage delta: `apply_patch_tests.rs` (+32 / −5) and `session/tests.rs` (+8 / −2) are the only test files touched, and `apply_patch_tests` is the only runtime-level test added. There's no new test in `tools/approval.rs` itself exercising the three `ApprovalDecisionSource` branches end-to-end. For a 526-line new module that owns approval routing, please add direct unit tests for `review_before_user_prompt` with stub hook + stub guardian to lock the precedence (hook-allow short-circuits, hook-deny short-circuits, hook-none falls through to guardian, guardian-none falls through to user).
- The deletion of 79 lines from `orchestrator.rs` and 95 from `sandboxing.rs` is welcome, but the diff doesn't show what's left in either file; reviewers should confirm those files are still internally coherent (no dead `use` statements, no orphan helper functions).
- **Question for author**: with `routes_approval_to_guardian(turn_context).then(new_guardian_review_id)` only generating an ID when guardian is enabled, what's the behavior if a hook returns `Allow` but a guardian is configured? Per the new precedence, the hook short-circuits and guardian never sees the request. That's a behavior change for users who configured guardian as a final stop-gap and may want it consulted regardless. Worth surfacing in the PR description.

## Verdict rationale

The architectural direction is correct and the deletions are the kind of consolidation this codebase needs. But the size, the security-critical surface, the `unreachable!()` panic in production code, the missing direct unit tests for the new module, and an arguable precedence change between hooks and guardian make this a **needs-discussion** rather than a merge. Address the `unreachable!()`, add module-level tests for the three source branches, and clarify the hook-vs-guardian precedence in the description, then this should be straightforward to land.
