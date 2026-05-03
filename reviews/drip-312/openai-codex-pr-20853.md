# openai/codex PR #20853 — [mcp-apps] Persist MCP Apps specific tool call end event

- URL: https://github.com/openai/codex/pull/20853
- Head SHA: `05cc15770b69afa478414c15283437ceb88f0b25`
- Verdict: **merge-as-is**

## Summary

A one-line policy patch in `codex-rs/rollout/src/policy.rs`: previously
all `EventMsg::McpToolCallEnd` events fell through to the default
non-persistence branch, so an MCP App's tool-call-end metadata was
discarded from the rollout log. This PR adds an explicit guard:

```rust
EventMsg::McpToolCallEnd(event) if event.mcp_app_resource_uri.is_some() => {
    Some(EventPersistenceMode::Limited)
}
```

So when (and only when) the tool call carries an `mcp_app_resource_uri`,
the event is persisted in `Limited` mode (i.e., the rollout writer
keeps the event but elides large payload fields per the existing
`Limited` policy).

## Specific references

- `codex-rs/rollout/src/policy.rs:115-122` — the new match arm. Sits
  inside `event_msg_persistence_mode`, immediately before the catch-all
  block that returns `None` for `EventMsg::Error | GuardianAssessment |
  WebSearchEnd | …`.
- The guard `event.mcp_app_resource_uri.is_some()` is the discriminator
  between an MCP App tool call and a generic MCP tool call. Generic MCP
  tool calls retain the prior `None` (don't persist) behavior.

## Commentary

This is the smallest correct fix for the stated problem. The
`event_msg_persistence_mode` function is already the canonical
allowlist for which event types reach disk, so adding one more arm
here is consistent with the established pattern (see the surrounding
arms for `WebSearchBegin`, `TaskStarted`, etc.).

The choice of `EventPersistenceMode::Limited` rather than `Full` is
defensible: MCP App resource bodies can be arbitrary blobs, and
`Limited` mode in this codebase strips large content while keeping
identifying metadata (URI, tool name, status). Anyone replaying a
rollout will see "an MCP App tool call ended for resource X" even if
the response body is gone, which is the right tradeoff.

Three things I checked and found OK:

1. **No regression for non-MCP-App tool calls.** The `if` guard means
   the existing behavior (return `None`, do not persist) is preserved
   for plain MCP tool calls. The only behavior change is for events
   where `mcp_app_resource_uri.is_some()`.

2. **No new fields, no schema migration.** `mcp_app_resource_uri` is
   already an `Option<String>` on the `McpToolCallEnd` event, so this
   patch only changes what we *persist*, not what we *emit*. Old
   rollout files remain readable.

3. **Match exhaustiveness.** Adding a guarded arm above the catch-all
   is the correct Rust idiom and the compiler will catch any future
   `EventMsg` variants that need explicit handling.

No tests in the diff is mildly disappointing — a snapshot test asserting
"event with `mcp_app_resource_uri = Some(_)` round-trips through the
rollout writer" would lock this in — but the change is small enough
that I wouldn't block on it. Merge as-is.
