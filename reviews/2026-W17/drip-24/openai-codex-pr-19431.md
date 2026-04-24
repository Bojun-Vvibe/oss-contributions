# openai/codex PR #19431 — Route opted-in MCP elicitations through Guardian

- **Repo:** openai/codex
- **PR:** [#19431](https://github.com/openai/codex/pull/19431)
- **Head SHA:** `f047565e00ead4d4c910a07b4cd562637f6e205a`
- **Author:** cd-oai (Clark DuVall)
- **Size:** +824/-76 across 24 files
- **Reviewer:** Bojun (drip-24)

## Summary

Adds a generic `ElicitationReviewer` callback hook on the MCP
connection manager so MCP elicitations carrying explicit
`codex_approval_reviewer: "guardian"` opt-in metadata get routed
through the Guardian approval path instead of bypassing it. The
motivating case is Browser Use origin-access prompts, which are
sent as MCP elicitations rather than direct tool-call approval
requests, so they previously skipped Guardian entirely.

The change is structurally clean — a new optional callback type
threaded through `McpConnectionManager::new_with_clients`, and four
callsites that pass `None` (snapshot/status/resource readers don't
need a reviewer).

## Key changes

### `codex-mcp/src/mcp_connection_manager.rs` (+~50)

New public types at lines 280–290:

```rust
pub struct ElicitationReviewRequest {
    pub server_name: String,
    pub request_id: RequestId,
    pub elicitation: CreateElicitationRequestParams,
}

pub type ElicitationReviewer = Arc<
    dyn Fn(ElicitationReviewRequest) -> BoxFuture<'static, Result<Option<ElicitationResponse>>>
        + Send + Sync,
>;
```

The reviewer returns `Result<Option<ElicitationResponse>>`:
- `Ok(Some(response))` → reviewer resolved it (Guardian decided);
- `Ok(None)` → reviewer abstained, fall through to the normal
  TUI/event flow;
- `Err(_)` → propagate failure.

The reviewer is invoked at line ~388, **after** the existing
auto-accept-empty-form-on-DangerFullAccess check at line 367 and
**after** the policy-rejection check. Order matters and is correct:
auto-accept stays a fast path that doesn't burn a Guardian request,
and policy-rejected elicitations don't get a chance to be
auto-approved by Guardian.

### Test coverage (`mcp_connection_manager_tests.rs`)

Four new/modified tests pin down the contract:

1. `full_access_auto_accepts_before_calling_elicitation_reviewer`
   — verifies the auto-accept fast path beats the reviewer (uses
   an `AtomicBool` flag and asserts `!called`).
2. `elicitation_reviewer_can_resolve_without_emitting_event` —
   reviewer returns `Some(Accept)`, asserts no event hits the
   channel (`rx_event.try_recv().is_err()`).
3. `elicitation_reviewer_none_preserves_event_flow` — reviewer
   returns `None`, asserts the elicitation event is still emitted
   and the responder map is populated.
4. The two pre-existing tests are updated to pass an explicit
   `/*reviewer*/ None` arg.

The first test is the one I most care about — the auto-accept fast
path being unconditional is load-bearing for performance, and a
future refactor that flipped the order would silently start
double-approving every empty-form elicitation through Guardian.

## Concerns

1. **`Arc<dyn Fn(...) -> BoxFuture<...> + Send + Sync>` reviewer
   shape is correct but cumbersome.**

   Every callsite that doesn't need a reviewer (4 of them in this
   PR — the snapshot readers in `codex-mcp/src/mcp/mod.rs` lines
   319/393/474) passes `/*reviewer*/ None`. That's fine, but the
   `BoxFuture` + double `Arc::clone` pattern in `make_sender` is
   verbose. A `trait ElicitationReviewer { async fn review(...) }`
   would be ergonomically cleaner; flagged as a follow-up rather
   than a blocker since the closure form is what the codex-core
   side wires up against `Guardian` directly.

2. **No metadata validation on `codex_approval_reviewer`.**

   The PR description says the reviewer "validates explicit
   `mcp_tool_call` opt-in metadata", but the validation lives in
   the codex-core reviewer impl (not visible in the diff snippet I
   can see). The connection manager itself doesn't inspect the
   `_meta` block — it just calls the reviewer. This is the right
   layering, but it means if the codex-core impl gets a typo (e.g.
   matches `"codex_approval_reviewer": "guardian"` case-sensitively
   when servers send `"Guardian"`), the elicitation silently falls
   through to the normal TUI flow with no warning. A
   debug-only `tracing::warn!` for "reviewer abstained on opt-in
   metadata that looked guardian-shaped" would help catch this
   class of bug.

3. **Guardian timeout/cancellation semantics aren't visible here.**

   The PR body says the reviewer "maps Guardian approval, denial,
   timeout, and cancellation decisions back to MCP elicitation
   responses" — that mapping happens in the codex-core impl. Worth
   double-checking that a Guardian timeout maps to
   `ElicitationAction::Cancel` rather than `Decline` (semantic
   difference: cancel = retryable, decline = persistent denial).
   Not blocking on this PR, but the codex-core companion change
   needs careful review.

4. **`ResponderMap` cleanup on reviewer-resolved elicitations.**

   When the reviewer resolves with `Some(response)`, the code
   returns the response directly without inserting an entry into
   `self.requests`. Good — no leak. But if a future change moves
   the responder-map insertion above the reviewer call (e.g. for
   Guardian-async-with-progress UX), the cleanup path needs to
   remove it. Worth a comment at the early-return site.

## Verdict

`merge-after-nits` — the design is correct, the test coverage is
exactly what I'd ask for (auto-accept-beats-reviewer pinned with
`AtomicBool`), and the four `None` callsites are well-marked. Two
minor follow-ups: a debug log when the codex-core reviewer
abstains on opt-in-shaped metadata (catches typos), and a comment
at the reviewer-resolved early-return explaining why the responder
map isn't touched.

The Guardian-side mapping is the higher-risk piece and lives
outside this PR — flagging for the companion review.

## What I learned

The `Result<Option<T>>` "reviewer abstains" pattern is a clean way
to retrofit a new approval path onto an existing event-emitting
flow without breaking the old path: callers that don't supply a
reviewer get identical behavior, and callers that do can fall
through on a per-request basis. Same shape as the SQLAlchemy
`event.contains` opt-out pattern, and the LSP server-capability
"this method is not implemented" fall-through. The lesson:
when adding a new gate to a hot path, prefer a callback that can
abstain over a flag that forces every request through the new
gate.
