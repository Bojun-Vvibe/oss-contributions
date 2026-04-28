# openai/codex#20069 — Bypass ARC for always-allow MCP tools in auto-review

- **Repo:** [openai/codex](https://github.com/openai/codex)
- **PR:** [#20069](https://github.com/openai/codex/pull/20069)
- **Head SHA:** `68c966f240d56adbb82b5993118325ba248f59cc`
- **Size:** +16 / -25 (two files: `codex-rs/core/src/mcp_tool_call.rs`, `codex-rs/core/src/mcp_tool_call_tests.rs`)
- **State:** OPEN

## Context

Two safety monitors compete for the same approval question: ARC (the
risk-classifier monitor) can return `ask-user` even for an MCP tool the
user has explicitly configured `approval_mode = approve` (always allow).
When `approvals_reviewer = auto_review`, the `ask-user` then routes into
guardian, which re-asks. End result: a user-explicit "always allow"
decision still routed through *both* monitors before the tool ran. The
explicit user decision was advisory rather than authoritative.

## Design analysis

Single short-circuit at `codex-rs/core/src/mcp_tool_call.rs:889-895`:

```rust
if approval_mode == AppToolApproval::Approve
    && turn_context.config.approvals_reviewer == ApprovalsReviewer::AutoReview
{
    return None;
}
```

Returns `None` from `maybe_request_mcp_tool_approval` — same shape as the
existing early-return at `:888` for the `Reject` case. Placed *before* the
`requires_mcp_tool_approval(annotations)` lookup at `:898`, so it
short-circuits both ARC and guardian on the always-allow path.

The conditional is correctly conjoined: it only fires when *both*
`approval_mode == Approve` (user explicitly opted in) AND
`approvals_reviewer == AutoReview` (this is an automated review session,
not a human-driven session that might want a confirmation). For non-auto-
review sessions, the existing ARC-then-guardian sequence still applies —
preserving the safety net for the cases where the user hasn't
explicitly waived it.

The companion test rename
`approve_mode_routes_arc_ask_user_to_guardian_when_guardian_reviewer_is_enabled`
→ `approve_mode_skips_arc_and_guardian_when_guardian_reviewer_is_enabled`
(`mcp_tool_call_tests.rs:2389`) updates the test name to match the new
contract. The mock setup is converted from "expect ARC + guardian to be
called" to "expect *neither* called":

```rust
Mock::given(method("POST"))
    .and(path("/v1/responses"))
    .respond_with(ResponseTemplate::new(200))
    .expect(0)
    .mount(&server)
    .await;
// ...
Mock::given(method("POST"))
    .and(path("/codex/safety/arc"))
    // ...
    .expect(0)
    .mount(&server)
    .await;
```

The `.expect(0)` on both mocks is the right way to assert "this code path
must not call out" — `wiremock` will fail the test if either endpoint
gets hit.

The terminal assertion `assert_eq!(decision, None);` (`:2475`) replaces
`assert_eq!(decision, Some(McpToolApprovalDecision::Accept));`. Both
shapes ultimately allow the tool to run, but `None` is the correct return
when the policy says "no approval question needed" rather than "approval
question was asked and accepted." Right semantics.

## Risks / nits

1. **Lost coverage of "ARC could still want to weigh in even on
   always-allow."** The previous test asserted that ARC was called, got
   `ask-user`, routed to guardian, and guardian accepted. That test
   exercised the full `arc → guardian` chaining. With the new short-
   circuit, no test in this file exercises the chain at all for the
   `approve + auto-review` combination — and that's intentional, but
   there's no replacement test for "what if `approvals_reviewer != AutoReview`
   and ARC returns `ask-user` for an `Approve` tool?" That used to be
   covered by the same test. The PR body claims `Preserved existing tests
   that verify ARC can still block always-allow MCP tools outside guardian
   auto-review mode` — verify there's a separate test for that case
   actually present and not just removed alongside this rename.

2. **Tight coupling to `ApprovalsReviewer::AutoReview` discriminant.**
   The short-circuit fires only on the `AutoReview` variant. If a future
   `ApprovalsReviewer` variant adds another reviewer flavor (e.g.,
   `BatchReview`), the short-circuit won't apply and we'd quietly
   regress to the old behavior. A `match` on the enum with explicit
   `_ => false` arms — and a comment naming the policy — would make the
   omission grep-able when the enum grows.

3. **Logging gap.** The short-circuit returns `None` silently. For a
   policy decision this load-bearing, an `info!` or `debug!` line
   ("skipping ARC+guardian for always-allow MCP tool {name} in auto-
   review session") would help users debug "why didn't my safety monitor
   fire?" without instrumenting code.

4. **No symmetric short-circuit for `AppToolApproval::Reject`.** The
   existing early-return at `:888` for the `Reject` case is presumably
   above this. Worth confirming the ordering: this PR's short-circuit
   only fires for `Approve`, and the symmetric "always reject" branch
   already short-circuits independently. If both paths are now
   short-circuiting, document the policy (annotation comment block above
   the function) so future maintainers don't accidentally re-introduce
   ARC into the explicit-decision paths.

## Verdict

**merge-after-nits.** The fix is correctly minimal — six added lines for
the short-circuit, the test is converted to assert the strengthened
contract. Pre-merge ask: confirm nit 1 (the "ARC can block always-allow
outside auto-review" case is covered by *another* test still present) and
add the policy comment block from nit 4. Nits 2 and 3 are follow-ups.

## What I learned

- A user's explicit "always allow" decision and an automated-reviewer
  session are *both* signals that the safety net should step aside.
  Either alone might justify keeping ARC in the loop; both together
  shift the policy from "safety net" to "annoying gate the user already
  consciously bypassed."
- The `Option<Decision>` return shape carries a useful semantic
  distinction: `None` = "no question to ask, proceed under policy" vs
  `Some(Accept)` = "question was asked and answered yes." Conflating
  them works at runtime but loses signal in logs and audits.
- Test mocks with `.expect(0)` are the cleanest way to assert "this code
  path must not call out" — better than asserting the absence of a side
  effect after the fact, because the mock fails fast and points at the
  exact boundary.
