# openai/codex #20853 — [mcp-apps] Persist MCP Apps specific tool call end event

- PR: https://github.com/openai/codex/pull/20853
- Author: mzeng-openai
- Head SHA: `05cc15770b69afa478414c15283437ceb88f0b25`
- Updated: 2026-05-03T07:24:12Z

## Summary
Three-line change to the rollout persistence policy: when an `McpToolCallEnd` event carries a non-None `mcp_app_resource_uri`, persist it with `EventPersistenceMode::Limited` so resumed sessions can re-render the MCP App UI. Otherwise the event continues to be dropped.

## Observations
- `codex-rs/rollout/src/policy.rs` line ~118: the new arm `EventMsg::McpToolCallEnd(event) if event.mcp_app_resource_uri.is_some() => Some(EventPersistenceMode::Limited)` is correct as written, but it relies on the existing fallthrough to keep the no-URI case dropped. That coupling is not obvious from the diff alone — please add a brief inline comment ("// no URI: not needed for resume, fall through to None") so the next contributor doesn't accidentally add a sibling arm that breaks the assumption.
- `EventPersistenceMode::Limited` is the right choice over `Full` (the mcp result body can be large), but the PR description doesn't quantify what `Limited` strips for this event type. Worth adding a one-line note in the PR body so reviewers can confirm the resume render has everything it needs (in particular, the resource URI itself must survive the Limited filter — confirm in `EventPersistenceMode::Limited` serialization).
- No test in the diff. The codex-rs rollout policy module typically has table-driven tests by `EventMsg` variant; adding one row for "McpToolCallEnd with URI → Limited" and one for "without URI → None" would lock the behavior cheaply and is in line with the file's existing style.
- Three lines of code, no cross-crate impact, low blast radius — safe to merge after the test add and the inline comment.

## Verdict
`merge-after-nits`
