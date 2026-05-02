# openai/codex #20756 — [codex] support PreToolUse allow and ask decisions

- **Head SHA:** `ccc920a06254bde1066640d6584ef1c2cf3235ca`
- **Files:** 27 files in `codex-rs/core/src/tools/**`, `codex-rs/core/src/hook_runtime.rs`, `codex-rs/core/src/mcp_tool_call.rs`, `codex-rs/core/src/unified_exec/**`, `codex-rs/hooks/src/{events/pre_tool_use.rs,engine/output_parser.rs,lib.rs}` (+584/-70)
- **Verdict:** `merge-after-nits`

## Rationale

Extends the existing PreToolUse hook contract from a binary block/allow into a tri-state `PreToolUsePermissionDecision` (`allow` / `ask` / `deny`), so a hook can short-circuit approval prompts (auto-approve a Bash command it trusts) or escalate (force a prompt for a tool the user normally allow-listed). The signature change at `hook_runtime.rs:144` — `Option<String>` → `Result<Option<PreToolUsePermissionDecision>, String>` — is the right shape: `Err` is the explicit-block path that carries the user-facing reason, and `Ok(Some(decision))` carries the override. Call sites in `tools/handlers/{shell,apply_patch,mcp,unified_exec}.rs` and the new `mcp_tool_call::maybe_request_mcp_tool_approval_with_pre_tool_use_decision` (`mcp_tool_call.rs:204`) plumb the decision into approval routing so it bypasses the prompt loop when the hook returned `allow`.

Nits: (1) the old `maybe_request_mcp_tool_approval` is now `#[cfg(test)]` only (`mcp_tool_call.rs:921`) — fine, but rename it to `_legacy` or move it under `mod tests` so a future reader doesn't think it's still on the production path; (2) the `#[allow(clippy::too_many_arguments)]` on `handle_mcp_tool_call` is a smell — the function now takes 8 args including the new `pre_tool_use_permission_decision: Option<PreToolUsePermissionDecision>` (`mcp_tool_call.rs:107`); bundle the per-call inputs (`call_id`, `tool_name`, `hook_tool_name`, `arguments`, `pre_tool_use_permission_decision`) into a `McpToolCallRequest` struct in a follow-up; (3) the new `guardian_tests.rs` and `tool_dispatch_trace_tests.rs` should include at least one test where a hook returns `ask` and the user *denies* — the current test names suggest happy-path coverage only.

Risk is medium: this touches the approval routing path for every tool call. The `#[allow(too_many_arguments)]` and the tri-state expansion both change function signatures across the tools registry, so any out-of-tree forks of codex-rs will need to update. The default behavior (no hook configured) is unchanged because `permission_decision` is `Option` and falls through to today's prompt logic.
