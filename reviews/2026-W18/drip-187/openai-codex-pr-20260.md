---
pr: openai/codex#20260
sha: 867054d8d2d6f6206878afbf8fc4da6159de8856
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(core): truncate large mcp tool outputs in rollouts

URL: https://github.com/openai/codex/pull/20260
Files: `codex-rs/core/src/mcp_tool_call.rs`, `codex-rs/core/src/mcp_tool_call_tests.rs`, `codex-rs/core/src/context_manager/{history.rs,mod.rs}`, `codex-rs/core/src/session/mod.rs`

## Context

`McpToolCallEnd` events are persisted to rollout storage and consumed by the
app-server to rebuild `ThreadItem::McpToolCall` for the UI. Before this PR
the `result: result.clone()` at `mcp_tool_call.rs:364` shipped the unbounded
`CallToolResult` straight through, so a single MCP tool returning multi-MB
in `content`, `structuredContent`, or `_meta` would balloon the rollout file
and slow every subsequent session-replay / app-server reconstruction.

## What changed

New `truncate_mcp_tool_result_for_event` at `mcp_tool_call.rs:654-696`:

- Serializes the result; if under `MCP_TOOL_CALL_EVENT_RESULT_MAX_BYTES`
  (= `DEFAULT_OUTPUT_BYTES_CAP` from `codex-utils-pty`), passes through.
- Otherwise, calls `truncate_text` on the full serialized JSON and returns a
  synthetic `CallToolResult` with a single `{type: "text", text: <preview>}`
  content item, dropping `structured_content` and `meta`. `is_error` is
  preserved.
- Error variant: `truncate_text` directly on the message string.

Both call sites that emit `McpToolCallEnd` are switched (line 365 for the
approved-call path, line 1909 for the skip path). The model-visible payload
path (`sanitize_mcp_tool_result_for_model`) is unchanged — this only bounds
the *event/rollout* copy.

Tests at `mcp_tool_call_tests.rs:813-878` verify three cases:
small result preserved verbatim, large result bounded to `<2× cap + 1024`
with `truncated` marker and `structured_content`/`meta` cleared, large error
bounded to `< cap + 1024`.

The history.rs change is purely visibility (`pub(crate)` for
`truncate_function_output_payload`) and the session/mod.rs change is a
matching import — neither alters behavior.

## What's good

- Correct separation of concerns: event/rollout payload bounded, model-
  visible payload still goes through the dedicated sanitizer with full
  fidelity for the *current* turn.
- Honest comment at lines 668-674 about the 2× JSON-escape inflation is
  exactly the kind of detail that prevents future "why is this 2× the
  budget" panic. The test assertion `serialized.len() < CAP * 2 + 1024`
  reflects the same accounting.
- Idempotent: calling `truncate_mcp_tool_result_for_event` on an already-
  truncated payload is a no-op (under-budget passthrough).

## Verdict

`merge-as-is` — surgical fix to a real rollout-bloat bug, both call sites
covered, tests cover small/large/error branches, model-facing semantics
preserved.
