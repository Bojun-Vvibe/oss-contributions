# Review: openai/codex #21302 — [codex] support hook input rewrites

- Carrier: `openai/codex`
- PR: #21302
- Head SHA: `a02986bf`
- Drip: 387

## Verdict
`needs-discussion` (nd)

## Summary
Substantial expansion of the PreToolUse / PermissionRequest hook contract to
let hooks **rewrite tool input** before execution, and to let hooks
**override the approval routing decision** (force-prompt or auto-allow,
bypassing the existing policy/guardian/auto-approve cascade). The mechanism
is implemented for the MCP tool-call path with a `MAX_HOOK_INPUT_REWRITES = 8`
loop guard.

Three new wire-protocol surfaces are added:
1. `FunctionCallError::UpdatedInput(serde_json::Value)` — a non-error variant
   used to short-circuit the function-call path back to the dispatcher with
   a rewritten input.
2. `PreToolUseHookResult::Continue { updated_input, permission_decision }` vs
   `Blocked(String)` — replaces the previous `Option<String>` return that
   conflated "no opinion" with "continue".
3. `PermissionRequestDecision::Allow { updated_input: Option<Value> }` —
   adds an optional rewritten input to the existing `Allow`/`Deny` enum so
   permission hooks can also rewrite (not just gate).

The MCP path (`mcp_tool_call.rs:197-340`) wraps the
`maybe_request_mcp_tool_approval` call site in a `loop { ... continue; }`
that re-prompts after each `AcceptWithUpdatedInput(updated_input)` decision,
bounded by `MAX_HOOK_INPUT_REWRITES` to prevent a malicious or buggy hook
from looping forever.

## Diff anchors
- `codex-rs/core/src/function_tool.rs:6-8` — new `UpdatedInput` variant.
  This is in the `FunctionCallError` enum but is semantically a *successful*
  outcome ("hook rewrote the input, retry"). The `#[error("tool input
  rewritten by hook")]` attribute is misleading — see Concerns #1.
- `codex-rs/core/src/hook_runtime.rs:13-25` — new `PreToolUseHookResult`
  enum and `MAX_HOOK_INPUT_REWRITES = 8`. The constant choice of 8 is
  unjustified in code; a comment explaining (e.g., "8 covers all known
  legitimate use cases — input normalization, env injection, path
  canonicalization — without giving a misbehaving hook chain enough
  iterations to cause user-visible latency") would help.
- `:151-209` — `run_pre_tool_use_hooks` now returns the typed
  `PreToolUseHookResult` instead of `Option<String>`. The fallback path
  when `should_block && block_reason == None` returns `Continue {
  updated_input: None, permission_decision: None }` which silently
  swallows a misconfigured hook (it asked to block but didn't say why).
  This is conservative but should at least emit a `warn!` line.
- `codex-rs/core/src/mcp_tool_call.rs:204-260` — the new approval-loop
  in `handle_mcp_tool_call`. Correctly:
  - Bounds rewrites at `MAX_HOOK_INPUT_REWRITES`.
  - Calls `notify_mcp_tool_call_skip(..., "hook input rewrite limit
    exceeded", /*already_started*/ true)` on overflow with the proper
    `already_started: true` flag so the UI shows a graceful skip rather
    than a phantom new tool-call event.
  - Uses `invocation.arguments = Some(updated_input); continue;` to
    re-enter the approval check with the new input — important because a
    rewritten input may now match a different policy rule.
- `:1053-1097` — `maybe_request_mcp_tool_approval` now takes
  `pre_tool_use_permission_decision: Option<&PreToolUsePermissionDecision>`.
  The `Allow` short-circuit returns `None` (skip approval entirely);
  the `Ask { reason }` sets `force_user_prompt = true` which is then
  threaded through every subsequent gate (policy auto-approve, session
  cache, persistent cache, permission-request hooks, guardian routing).
  This is the load-bearing escape hatch — hooks can effectively veto
  every other approval mechanism and force the user prompt.
- `:1117-1175` — the `PermissionRequestDecision::Allow { updated_input }`
  match arm. Notably, when `updated_input == permission_request_tool_input`,
  the code returns `McpToolApprovalDecision::Accept` rather than
  `AcceptWithUpdatedInput` — correct optimization to avoid an unnecessary
  loop iteration.

## Concerns / discussion items
1. **`FunctionCallError::UpdatedInput` is a non-error in an error enum.**
   This is a code smell. Either rename the enum to `FunctionCallOutcome`,
   or extract `UpdatedInput` into a separate `Result<Outcome,
   FunctionCallError>` shape. The current shape will trip up future
   maintainers who pattern-match on `FunctionCallError` expecting only
   error variants.
2. **No security model documented for input rewriting.** A hook running
   under the user's permissions can now rewrite an MCP tool's arguments
   between the time the model proposes them and the time the tool
   executes. This is a powerful capability — for legitimate uses (path
   canonicalization, redacting secrets, enforcing org-wide guardrails)
   it's exactly what's wanted; for compromised hooks it's a confused-
   deputy attack vector (the user sees the model's proposed args in the
   approval UI, then the tool runs with different args). The PR body
   should explicitly enumerate: (a) is the rewritten input shown to the
   user before execution? (b) is there an audit log of rewrites? (c) what
   prevents a hook from rewriting `rm -rf ~/safe` to `rm -rf /`?
3. **`MAX_HOOK_INPUT_REWRITES = 8` is a magic number.** Should be either
   configurable (so paranoid orgs can set it to 1) or thoroughly
   justified in code comments. As-is, an attacker with hook-write access
   gets up to 8 input-rewrite cycles per tool call to obfuscate their
   final payload from any logging that fires only on the first attempt.
4. **`force_user_prompt` short-circuits the persistent approval cache.**
   This is correct (a hook saying "ask the user" should always ask), but
   it means a user can't make a "remember this decision" choice survive
   a hook upgrade that adds an `Ask` decision. Worth an explicit UX
   comment in the approval prompt ("this choice cannot be remembered
   because a policy hook is requesting confirmation").
5. **Test coverage in `mcp_tool_call_tests.rs` is signature-only.** The
   visible diff additions are all `/*pre_tool_use_permission_decision*/
   None` — i.e., the existing tests are updated to compile with the new
   signature but no new test exercises the `Allow`-with-rewrite path,
   the `Ask`-forces-prompt path, the `Allow`-skips-cache path, or the
   `MAX_HOOK_INPUT_REWRITES` overflow path. Each of these is a separate
   contract that could regress silently.
6. **`PermissionRequestDecision::Allow` is now a struct variant.** This
   is a breaking change to the `codex-hooks` crate's public API. Any
   downstream hook author using `PermissionRequestDecision::Allow` as a
   unit variant (e.g., in a `match` pattern) will hit a hard compile
   error. The PR body should call out which downstream crates were
   audited and whether a deprecation window was considered.
7. **The `pre_tool_use_permission_decision` parameter is plumbed through
   `handle_mcp_tool_call → maybe_request_mcp_tool_approval` but the
   `clippy::too_many_arguments` `#[expect]` attribute is added at both
   sites with the same justification.** This is pragmatic but the right
   long-term fix is a `McpToolApprovalContext` struct bundling the
   invocation/policy/hook state.

## Risk
High. This is a meaningful expansion of trust placed in hooks (input
rewriting + approval-routing override) wired through the MCP tool-call
path. The implementation is careful (loop bound, audit-friendly skip
notification, `force_user_prompt` flag propagation) but the security model
needs explicit maintainer ack and the test surface needs to grow before
this can land. Recommend splitting into (a) the type-system refactor
(rename `FunctionCallError`, struct-variant `Allow`) which is mechanical,
and (b) the actual rewrite-and-route capability with full test coverage
and a security-model doc.
