# openai/codex PR #20804 — [codex] type approval request families

- Head SHA: `2c88eceb55724810d3e06f76a685d416a737ca67`
- Size: +402 / -394, 19 files (Rust)

## Specific refs

- `codex-rs/core/src/approval_request.rs` (renamed from `core/src/guardian/approval_request.rs`) — top-level re-home of the canonical `ApprovalRequest` so it's no longer guardian-scoped (`approval_request.rs:34-95`). New typed structs: `CommandApprovalRequest`, `ApplyPatchApprovalRequest`, `McpToolCallApprovalRequest`, `RequestPermissionsApprovalRequest`.
- `core/src/approval_request.rs:96-105` — `permission_request_payload()` is now a dispatcher across the four families; only `RequestPermissions` returns `None`. Per-family impls return non-Option payloads, removing one layer of `unwrap()`/`unreachable!()` ergonomics.
- `core/src/session/mod.rs:+12/-18` — session prompt API now takes `CommandApprovalRequest` or `ApplyPatchApprovalRequest` directly. This is the headline win: invalid family→prompt routing no longer compiles.
- `core/src/tools/runtimes/shell.rs:+8/-8`, `apply_patch.rs:+7/-6`, `unified_exec.rs:+8/-11`, `network_approval.rs:+6/-9` — call sites updated to construct the typed family request.
- `core/src/guardian/tests.rs:+63/-53` — tests rewritten against the new types; named per-test (`approval_elicitation_request_uses_message_override_…`, `handle_exec_approval_uses_call_id_for_guardian_review_and_approval_id_for_reply`).

## Assessment

Solid type-safety win — moves runtime `unreachable!()` invariants into the type system at the prompt boundary. The rename out of `guardian/` is correct: this enum is shared by hooks, guardian review, and user prompts, so `guardian/` was misleading. Mechanical conversion at call sites; risk surface is mostly "did we miss a variant" which the compiler now answers.

Minor: PR description notes a pre-existing stack-overflow in `tools::handlers::multi_agents::tests::tool_handlers_cascade_close_and_resume_…` blocks `cargo test -p codex-core`. Not introduced here, but reviewers shouldn't be asked to verify unrelated panics.

verdict: merge-after-nits