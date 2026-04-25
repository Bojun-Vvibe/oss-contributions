# PR #19467 — feat: route MCP elicitations through guardian review

- Repo: openai/codex
- Head SHA: 6587e32735df9ae7c1b6858e49d5a90275a3ebe4
- Files touched: 23

## Summary

This is the productionized follow-up to the prototype landed in #19431
(reviewed in drip-24). The protocol surface gains a fourth Guardian
review action shape — `McpElicitationGuardianApprovalReviewAction`
(carries `serverName`, `message`, optional `connectorId` /
`connectorName`) — registered on `ServerNotification.json` and the v2
`ItemGuardianApprovalReview{Started,Completed}Notification.json`
schemas. Routing inside `codex-rs/core/src/session/mcp.rs` is what
makes this live: `Session::mcp_elicitation_reviewer()` now downgrades
to `Arc::downgrade(self)` and returns a `McpElicitationReviewer`
closure that, on each elicitation, gates on
`guardian_can_review_mcp_elicitation(&request.request)` (form-only,
URL elicitations still go to the user prompt path), pulls the active
turn's `cancellation_token`, and feeds the request into
`spawn_approval_request_review` with
`GuardianApprovalRequestSource::MainTurn`. The `tokio::select!` is
biased toward cancellation and falls back to `ReviewDecision::Denied`
when the channel drops.

## Specific findings

- `codex-rs/core/src/guardian/approval_request.rs` adds the
  `GuardianApprovalRequest::McpElicitation { id, turn_id, server_name,
  request, connector_id, connector_name }` variant, plus matching
  arms in `guardian_approval_request_to_json`,
  `guardian_assessment_action`, `guardian_reviewed_action`,
  `guardian_request_target_item_id` (returns `None` — elicitations
  have no target item id), and `guardian_request_turn_id`. The fact
  that target-item-id is `None` keeps elicitations off the
  per-item-completion notification join, which lines up with their
  out-of-band-turn nature; analytics still get the full
  `GuardianReviewedAction::McpElicitation` shape via
  `codex-rs/analytics/src/events.rs`.
- `codex-rs/core/src/guardian/tests.rs:728+` adds
  `guardian_approval_request_to_json_renders_mcp_elicitation_shape`,
  which pins the JSON envelope (`tool: "mcp_elicitation"`,
  `server_name`, nested `request` with `mode: "form"` and the verbatim
  `_meta` block — `connector_id`, `connector_name`, `origin`), and
  asserts both `guardian_request_target_item_id` returning `None` and
  `guardian_request_turn_id` falling through to the request's
  `turn_id` when a fallback is provided. That's the right shape of
  test for a serialized contract that downstream consumers will
  match on.
- `codex-rs/codex-mcp/src/mcp_connection_manager.rs` plumbs an
  `Option<McpElicitationReviewer>` through every `start_mcp_servers`
  call site (incl. `core/src/connectors.rs:279` and
  `core/src/mcp_skill_dependencies.rs` — the latter required
  promoting `&Session` to `&Arc<Session>` so the reviewer closure can
  hold a `Weak<Session>`).
- The `tokio::select!` block in `review_mcp_elicitation` is `biased`
  with cancellation first → `decision` second. That's right for
  elicitations: a turn cancel should always pre-empt a pending
  Guardian review.

## Risks / nits

- `guardian_request_target_item_id` returning `None` means UIs that
  key Guardian review notifications by item id won't see start /
  complete notifications matched to elicitations. The v2 notification
  shapes do carry the new action, so consumers can switch to keying
  off the action discriminant — worth calling that out in the PR
  body.
- `mcp_elicitation_reviewer` upgrades a `Weak<Session>` per call.
  When the session has been dropped between the elicitation arrival
  and the review attempt, the closure resolves to `None` and the
  caller will fall through to whatever default the elicitation path
  uses. That looks correct from the call-site code I see, but a
  comment in `session/mcp.rs:5+` would make the contract obvious to
  the next reader.
- `guardian_can_review_mcp_elicitation` is referenced but lives
  outside the visible diff window. Reviewer should confirm it
  rejects URL elicitations (per the PR notes) and any
  `requested_schema` that isn't structurally an empty object — if
  forms with required fields slip through, Guardian would have to
  invent a payload it has no way to know.

## Verdict

**merge-after-nits** — The protocol shape, schema regeneration, and
test coverage line up cleanly with the existing Guardian
infrastructure landed in drip-24. Resolve the
`Weak<Session>`-on-drop comment and the empty-form gate guarantee
before landing.

## What I learned

When introducing a new approval-shape variant in a tagged-union
protocol, the cheapest correctness gate is a JSON-shape pin in the
guardian crate's tests. Pinning `tool`, the action discriminant, and
the `_meta` passthrough catches every refactor that would otherwise
silently re-key the payload and break downstream consumers. Pairing
that with explicit assertions on
`guardian_request_target_item_id` / `guardian_request_turn_id`
locks in the routing-side contract too.
