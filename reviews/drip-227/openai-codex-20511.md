---
pr: openai/codex#20511
title: "[codex] Remove unused event messages"
head_sha: f794427f5d87e48df7f1736f1f521a3ef5590936
verdict: merge-after-nits
drip: drip-227
---

## Context

Several `EventMsg` variants and the `Op::Undo` request remained in the
protocol even though clients had migrated to item/lifecycle events and
the `Undo` task had been degraded to an "unavailable" shim. This PR
deletes the dead variants — `AgentMessageDelta`,
`AgentReasoningDelta`, `AgentReasoningRawContentDelta`,
`BackgroundEvent`, `UndoStarted`, `UndoCompleted` — plus the entire
`UndoTask` and `Op::Undo` codepaths. `McpStartupComplete`,
`WebSearchBegin`, and `ImageGenerationBegin` are intentionally kept;
the body explicitly notes the consumers that still need them.

## Diff walkthrough

The protocol-level deletions are at `protocol/src/protocol.rs:777-782`
(removes the legacy `Op::Undo` variant including its docstring noting
that it had degraded to an unavailable shim), `:908` (removes the
`"undo"` op name), `:1361-1370` (removes `AgentMessageDelta` and
`AgentReasoningDelta` variants from `EventMsg`), and `:3153-3166`
(removes `UndoStartedEvent` and `UndoCompletedEvent` payload structs).

Downstream match arms are pruned in lockstep across
`rollout/src/policy.rs:104,139-141,152,159,170-181`,
`rollout-trace/src/protocol_event.rs:235,237,260,309,311,335`,
`app-server-protocol/src/protocol/thread_history.rs:220` (drop the
`EventMsg::UndoCompleted(_) => {}` no-op arm), and
`core/src/codex_delegate.rs:265-268` (drop the "ignore all legacy
delta events" arm). Note that `policy.rs:152` and `:159` reorder
`WebSearchBegin` and `ImageGenerationBegin` so they remain in the
`None` (don't-persist) arm — consistent with the body's "kept for
in-progress item rendering" rationale.

The `UndoTask` code is deleted wholesale: `core/src/tasks/undo.rs`
goes to `/dev/null` (`:244-246` of the diff, 72 lines removed),
`tasks/mod.rs:5-7,57-60` removes the module declaration and re-export,
`session/handlers.rs:30,650-652` removes the `undo` async entry point
and its `submission_loop` arm at `:1117-1120`, and the TUI side at
`tui/src/chatwidget/slash_dispatch.rs:319-321` and
`tui/src/slash_command.rs:42,86,172` removes the
already-commented-out `SlashCommand::Undo` references (cleanup of
prior partial-removal scaffolding).

The compaction path also drops the dormant `truncated_count` background
notification at `core/src/compact.rs:168,201-209,211` since the
`BackgroundEvent` rendering path is gone — this is a behavior change,
not just a cleanup, but the user-visible effect was already a single
post-compaction info banner that nobody read.

## Risks / nits

- The `BackgroundEvent` removal also drops two MCP OAuth user-facing
  hints at `mcp_skill_dependencies.rs:138-145,158-165` ("Authenticating
  MCP {name}... Follow instructions in your browser if prompted." and
  the scope-retry banner). These were genuine UX affordances on the
  OAuth happy path; users now get silence during a browser-redirect
  step. Worth either logging at `info!` or moving to an `EventMsg`
  variant that still has a renderer.
- The "rebalance" of `WebSearchBegin`/`ImageGenerationBegin` in
  `rollout/src/policy.rs` shuffles the `None` arms; the test surface
  for `event_msg_persistence_mode` doesn't appear to assert these by
  name, so a future regression where one of them silently moves into a
  persisted arm wouldn't be caught.

## Verdict

**merge-after-nits** — net 460 lines deleted with consistent,
mechanical fan-out across protocol/rollout/TUI. The two MCP OAuth
hints are the only behavior loss worth a follow-up; the rest is dead
code that was correctly identified as such.
