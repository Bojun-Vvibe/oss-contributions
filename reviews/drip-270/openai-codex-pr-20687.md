# openai/codex #20687 — [codex] Split tool handlers by tool name

- URL: https://github.com/openai/codex/pull/20687
- Head SHA: `5b91e5f28c9fba32792a81bb6cba5b9c08bc0b2b`
- Author: @pakrym-oai
- Stats: +1325 / -896 across 44 files

## Summary

Refactor: the omnibus `BatchJobHandler` / `GoalHandler` / `UnifiedExecHandler`
that dispatched on `tool_name.name.as_str()` are split into one handler
per public tool name (`SpawnAgentsOnCsvHandler`,
`ReportAgentJobResultHandler`, `CreateGoalHandler`, `UpdateGoalHandler`,
`ExecCommandHandler`, etc.). Every `ToolHandler` impl now declares its
`tool_name() -> ToolName`, removing the in-handler string-matching arms.

## Specific feedback

- `codex-rs/core/src/tools/handlers/agent_jobs.rs:39-41` — the old
  `BatchJobHandler` is now two handlers with their own `tool_name()`
  declarations (`spawn_agents_on_csv`, `report_agent_job_result`).
  This is the right move: the `match tool_name.name.as_str()` arm
  (deleted around line 134) was the classic "stringly-typed dispatch
  inside a dispatch" smell.
- `codex-rs/core/src/tools/code_mode/execute_handler.rs:80-83` and
  `code_mode/wait_handler.rs:43-46` — the new `tool_name()` returns
  `ToolName::plain(PUBLIC_TOOL_NAME)` / `WAIT_TOOL_NAME` from the same
  constant the registry uses. This keeps the handler<->registry binding
  in lockstep; nice.
- `codex-rs/core/src/session/tests.rs:64-71` and `:8140-8295` — the
  test refactor splits `let handler = GoalHandler` into separate
  `CreateGoalHandler` / `UpdateGoalHandler` instances on each test.
  This makes the tests more honest about which handler they're
  actually exercising. Confirm the original `GoalHandler` struct is now
  removed (not just unused) so no caller can pick it up by mistake.
- `codex-rs/core/src/session/tests/guardian_tests.rs:498-501` —
  `UnifiedExecHandler` → `ExecCommandHandler` rename is mechanical.
  Make sure any external consumer (other crates, plugin entry points,
  or PyO3/wasm bindings) is not still importing the old struct name.
- `codex-rs/core/src/tools/handlers/shell.rs` (+236/-72) and
  `unified_exec.rs` (+262/-224) are the largest semantic deltas in
  this otherwise mechanical PR. The diff is truncated in this review
  surface — the reviewer should sanity-check that the new shell handler
  preserves the existing approval/sandboxing surface (no quiet
  permission-policy regressions during the split).
- `codex-rs/core/src/tools/registry.rs:7-29 deletions` — the central
  router shrinks because each handler now self-identifies. Verify the
  registry still rejects duplicate `tool_name()` registrations at
  startup; otherwise two handlers claiming the same name silently
  overwrite each other.
- `codex-rs/core/src/tools/handlers/mcp_resource.rs` (+297/-282) is a
  near-total rewrite, not just a split. Worth calling out separately
  in the PR description so reviewers don't gloss over it as "mechanical
  too."
- No new tests beyond the rename — at this scope, an end-to-end
  smoke test that walks one tool of each `kind` would catch any
  registry-binding regression.

## Verdict

`merge-after-nits` — direction is clearly correct (kills stringly-typed
dispatch, makes ownership obvious), but the PR is large enough that the
non-mechanical files (`shell.rs`, `unified_exec.rs`, `mcp_resource.rs`)
deserve to be flagged in the description and ideally split into
follow-ups. Adding a registry-startup duplicate-name guard would lock
in the invariant the refactor relies on.
