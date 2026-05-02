# openai/codex PR #20737 — [codex] centralize approval routing

- **URL**: https://github.com/anomalyco/codex/pull/20737
- **Head SHA**: `d0d7aacaee35`
- **Files touched**: `codex-rs/core/src/mcp_tool_call.rs`, `codex-rs/core/src/session/mod.rs`, `codex-rs/core/src/session/tests.rs`, `codex-rs/core/src/state/service.rs`, plus new `tools/approval` module

## Summary

Refactors the MCP tool-call approval path so it no longer hand-rolls the "permission hooks → guardian → user prompt" sequence inline. Introduces an `ApprovalRequest` value object and a `review_before_user_prompt(...)` helper that returns a single `ApprovalOutcome { decision, source, rejection_message }`. The MCP path becomes a `match` on `outcome.source`. `ApprovalStore` moves from `tools::sandboxing` to `tools::approval`.

## Comments

- `mcp_tool_call.rs:67-83` — `review_before_user_prompt` is invoked with `evaluate_permission_request_hooks: true` as a `bool` flag. This is exactly the kind of arg that turns into a footgun later (someone adds `evaluate_guardian: bool` next to it and now every callsite is `(true, false)`). Prefer a small `ReviewPolicy` enum or struct.
- `mcp_tool_call.rs:149-151` — `unreachable!("review_before_user_prompt never returns user-prompt outcomes")` is load-bearing on the function name. Add a `debug_assert` or, better, encode it in the type: split `ApprovalDecisionSource` into `PreUserSource` (no `User` variant) and have the function return that narrower type.
- `mcp_tool_call.rs:119-148` — when `source == PermissionRequestHook` we return `McpToolApprovalDecision::Accept` for `Approved | ApprovedExecpolicyAmendment | ApprovedForSession | NetworkPolicyAmendment`. Previously the permission-hook path only knew Allow/Deny — collapsing the four guardian-style variants into Accept means any execpolicy amendment carried by the hook is silently dropped. Verify that hooks never return amendment variants today, and add a test that locks that in.
- `session/tests.rs:184-191` — the test mock returns `ApprovalDecisionSource::User` from a function whose name suggests "before user". This works because the mock is the trait impl, not `review_before_user_prompt`, but it's a confusing read; add a comment.
- Module move `tools::sandboxing::ApprovalStore` → `tools::approval::ApprovalStore` (`session/mod.rs:164,170`) is a clean re-org but breaks any out-of-tree consumers that imported the old path. Note in CHANGELOG / release notes if the crate is published.
- `metadata.and_then(|metadata| metadata.connector_id.clone())` patterns repeat 5 times (lines ~51-58). A `From<&Metadata>` for the annotation block would compress this nicely.

## Verdict

`merge-after-nits` — the centralization is a clear win; the four variant-collapse and bool-flag concerns deserve resolution but none block landing.
