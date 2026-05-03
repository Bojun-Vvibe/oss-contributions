# openai/codex PR #20853 — [mcp-apps] Persist MCP Apps specific tool call end event

- Head SHA: `05cc15770b69afa478414c15283437ceb88f0b25`
- Files changed:
  - `codex-rs/rollout/src/policy.rs` (+3)

## Analysis

Single-arm extension to `event_msg_persistence_mode` so that an `EventMsg::McpToolCallEnd` whose payload carries an `mcp_app_resource_uri` is now persisted with `EventPersistenceMode::Limited` instead of being dropped.

Diff at `codex-rs/rollout/src/policy.rs:115-120`:
```
+        EventMsg::McpToolCallEnd(event) if event.mcp_app_resource_uri.is_some() => {
+            Some(EventPersistenceMode::Limited)
+        }
```

This sits inside the wildcard tail of `event_msg_persistence_mode`, between the existing arm that returns `None` for plain events and the explicit `EventMsg::Error | EventMsg::GuardianAssessment | EventMsg::WebSearchEnd | …` group. The pattern guard `if event.mcp_app_resource_uri.is_some()` keeps the change surgical: only MCP App-flavored tool-call ends trigger persistence, ordinary MCP tool calls retain whatever the previous default was.

Why this matters: rollout replays use `EventPersistenceMode` to decide what to write to disk and what to replay back to clients. Without persisting these ends, a Codex App resume cannot reconstruct the MCP App pane (the App rendering depends on having the resource URI from the end event). `Limited` (rather than `Full`) suggests the replay surface is intentionally constrained — likely just enough to re-mount the MCP App pane without re-emitting any large payload bodies.

Risks:
- The arm relies on `mcp_app_resource_uri` being a stable signal that the event is App-bound. If a producer ever sets that field on a non-App tool call, those events would also be persisted. Seems unlikely to be a problem in practice, but a doc comment on the field at its definition site would harden this against future drift.
- No new test in this diff. A regression test that constructs both variants (with and without `mcp_app_resource_uri`) and asserts the persistence mode would be cheap and is usually expected for `policy.rs` changes — most existing arms there are covered.

## Verdict

`merge-after-nits`

## Nits

- Add a unit test in the same file (or its sibling `policy_tests.rs` / `tests/`) covering both branches of the new guard.
- Doc comment on `McpToolCallEndEvent::mcp_app_resource_uri` clarifying the persistence contract.
