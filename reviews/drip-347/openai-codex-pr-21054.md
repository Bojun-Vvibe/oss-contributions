# openai/codex#21054 — rollout: store web search and mcp tool calls

- **Head SHA**: `581a7e09e709796a0789ae0259371e4c35424a51`
- **Verdict**: `merge-as-is`

## Summary

Single-file change to `codex-rs/rollout/src/policy.rs` (+2/-5). Promotes `WebSearchEnd` and the unconditional `McpToolCallEnd` arm from the dropped/conditional bucket up into the `EventPersistenceMode::Limited` bucket. PR body: "Codex App would like these."

## Findings

- `codex-rs/rollout/src/policy.rs:104-110`: `McpToolCallEnd(_)` and `WebSearchEnd(_)` are added to the unconditional `Limited` arm.
- `:120-125`: the previous gated branch — `EventMsg::McpToolCallEnd(event) if event.mcp_app_resource_uri.is_some() => Some(EventPersistenceMode::Limited)` — is removed. That gate was only persisting MCP tool-call ends when an app resource URI was present; everything else was being tossed. The new behavior persists all MCP tool-call ends in `Limited` form, which matches the surrounding policy for `ContextCompacted`, `TurnComplete`, etc.
- `:127-133`: `WebSearchEnd(_)` is removed from the `None` (drop) arm and `ExecCommandEnd(_)` / `ViewImageToolCall(_)` etc. remain dropped — consistent intent. The asymmetry between `ExecCommandEnd` (still dropped) and `McpToolCallEnd` (now persisted) is worth a one-liner comment for future readers, but not a blocker.

## Recommendation

Trivial, scoped change with the right semantics for downstream rollout consumers that need to reconstruct tool-call timelines. No new APIs, no tests required given the existing `policy.rs` test coverage will exercise these arms. Land it.
