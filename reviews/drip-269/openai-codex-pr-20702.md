# openai/codex #20702 — Support PreToolUse approvalDecisions

- URL: https://github.com/openai/codex/pull/20702
- Head SHA: `165f0f2f6472f9408d3a9e6325f2b5f2c108c6c6`
- Author: @abhinav-oai
- Stats: +1015 / -181 across 21 files

## Summary

Adds a narrow approval-control surface to `PreToolUse` hooks: a hook can
emit `permissionDecision: "allow" | "ask"` (with `"defer"` reserved-but-
unimplemented), recorded turn-locally keyed by `tool_use_id`, and consumed
by a single shared approval router across shell, unified exec, apply patch,
network approval, execve escalation, and MCP approval paths. Critically,
`ask` pierces both normal and strict auto-review and bypasses cached
session approvals so it always reaches a human.

## Specific feedback

- `codex-rs/core/src/hook_runtime.rs:153-176` — `tool_use_id.clone()` plus
  the new `permission_decision` plumbing into
  `pre_tool_use_approval_overrides.record(tool_use_id, permission_decision)`
  is the right pattern: state lives in turn context, not hook output. This
  cleanly avoids threading the decision through every approval call site.
- `codex-rs/core/src/mcp_tool_call.rs:957-1057` — the refactor from "early
  return on auto-approval" to "compute `RoutingApprovalRequirement`, then
  run `resolve_approval_route(...)`" is the single shared approval router
  the PR body promises. Behavior preserved for the no-hook case, and the
  `pre_tool_use_permission_decision.is_none()` guards correctly skip
  remembered-approval lookups when the hook has already steered the
  decision (so `ask` actually pierces).
- The `routes_approval_to_guardian(turn_context)` argument plumbed into
  `resolve_approval_route` keeps Guardian/strict-auto-review semantics
  intact; combined with the PR body claim that "strict auto review stays
  stronger than `allow`", the precedence ordering is sensible.
- Reserving `permissionDecision: "defer"` in the schema while keeping it
  unimplemented is a deliberate forward-compat hook for Codex exec — fine,
  but worth either a `#[serde(other)]` arm or an explicit error path so a
  hook that emits `"defer"` today gets a clear "not yet supported" error
  rather than being silently coerced.
- Test surface listed in PR body is broad: codex-hooks lib, codex-core
  approval_routing/pre_tool_use_approval_overrides/mcp_tool_call/
  network_approval/unix_escalation, plus integration tests
  `pre_tool_use_allow_skips_shell_command_approval` and
  `pre_tool_use_ask_prompts_user_with_hook_reason`. That's the right
  fan-out for a change this wide.
- `permissionDecisionReason` going to the user without entering
  model-visible context is the right default — keeps the model from
  conditioning on user-facing rationale strings.
- Public Codex hooks docs are explicitly NOT updated in this PR (per "Docs"
  section). That's fine for a foundation PR, but the follow-up doc PR
  needs to land before this is exposed broadly, otherwise users will be
  configuring `permissionDecision` based on changelog spelunking.

## Verdict

`needs-discussion` — the design is sound and the test coverage is
serious, but: (1) the `defer` reserved-state error contract should be
nailed down before merge, and (2) the PR body explicitly defers the docs
PR. Both deserve a quick maintainer sync rather than a silent merge.
