# openai/codex #20702 — Support PreToolUse approval decisions

- **Repo:** openai/codex
- **PR:** https://github.com/openai/codex/pull/20702
- **HEAD SHA:** `30a05d7`
- **Author:** abhinav-oai
- **Verdict:** `merge-after-nits`

## What the diff does

Extends the `PreToolUse` hook protocol so a hook can not just block a tool call but actively *steer* the permission boundary for it. Hooks may now emit `permissionDecision: "allow" | "ask"` (with `permissionDecisionReason`) in their `hookSpecificOutput`; `"defer"` is reserved in the JSON schema (`codex-rs/protocol-ts/.../*.json:1967` area) for a future Codex exec integration but explicitly unimplemented today.

The wire→core decode lives in `codex-rs/hooks/src/engine/output_parser.rs:19-32,132-141`: `PreToolUseOutput` gains an `Option<PreToolUsePermissionDecision>` field, and the parser only materializes it when `invalid_reason.is_none()` (correct — a malformed hook payload should not be allowed to grant or demand approvals through the side door). The new enum carries the optional `reason` so the user-facing prompt can render the hook's justification.

The load-bearing routing change is the new `resolve_approval_route` (`codex-rs/core/.../approval.rs:418-440`):

- `Some(Ask { reason })` → unconditional `ApprovalRoute::PromptUser { reason, cache_policy: BypassCachedApprovals }` — bypasses *any* prior cached "always allow" decision the user had set, which is exactly right when a hook is asking specifically for *this* call.
- `Some(Allow { .. })` while `!strict_auto_review` → `ApprovalRoute::Skip`.
- `Some(Allow { .. })` while `strict_auto_review` → falls through to `RouteToGuardian` — i.e. a hook cannot waive auto-review's strict mode, only the guardian can. This asymmetry is the right policy.
- `None` → unchanged legacy path.

Plumbed through `PreToolUsePayload` and the per-tool dispatchers (`session/mod.rs:606,726,734,796`, `mcp_tool_call.rs:1400,1454`) so the decision actually reaches the approval gate.

## Why it's right

- The four unit tests at `:455-510` lock the four interesting cells of the truth table: `Ask` while strict, `Ask` while non-strict, `Allow` while non-strict (skip), and `Allow` while strict (still routed). Without the strict-mode test the asymmetry would be hidden behind a default flip.
- `BypassCachedApprovals` on `Ask` is the right cache policy — a user who previously clicked "always allow `bash`" should still see the prompt when a hook later decides this *specific* invocation needs human eyes.
- The wire→core enum split (`PreToolUsePermissionDecisionWire` → `PreToolUsePermissionDecision`) keeps unsupported variants (`defer`) representable on the wire without leaking into the routing match, and the now-unified `Allow | Ask` arm in `unsupported_pre_tool_use_hook_specific_output` (line ~357) correctly stops emitting the "unsupported" warning for the variants we now handle.
- Reserving `defer` in the JSON schema with a `description` naming the Codex exec integration is the right shape — clients can already ship the symbol while Codex no-ops it.

## Nits

1. `PreToolUsePayload` constructions at `session/mod.rs:726,734,796` repeat the same struct-literal pattern; a single helper builder (or `PreToolUsePayload::for_tool(...)`) would prevent one-of-three-sites drift later.
2. `permissionDecisionReason` is intentionally not added to model-visible context — worth a one-line comment at the redaction site so a future "let's surface hook reasons everywhere" change doesn't silently leak it back to the model.
3. The `defer` schema variant has no negative test asserting it is rejected at parse-time today; without that, a future enum-add could silently start accepting it.
4. `ApprovalCachePolicy::BypassCachedApprovals` should grow a doc comment naming the `Ask`-from-hook case as the canonical example so the policy stays load-bearing across refactors.
5. PR description uses `"defer"` but the schema enum at line ~1990 is single-variant — confirm the long-term plan is multi-decision (`defer | allow | ask`) and not just a placeholder, otherwise the branch-of-one is over-engineered.

`merge-after-nits` — right primitive (decision enum carried end-to-end, asymmetric strict-mode policy, cache-bypass on `Ask`), tests cover the truth table, but the multi-site payload construction and missing reject-test on `defer` want follow-ups before merge.
