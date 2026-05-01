# openai/codex #20687 — [codex] Split tool handlers by tool name

- **Repo:** openai/codex
- **PR:** https://github.com/openai/codex/pull/20687
- **HEAD SHA:** `a435dd9`
- **Author:** pakrym-oai
- **Verdict:** `merge-after-nits`

## What the diff does

Refactors the tool-handler registry so each registered handler instance owns *exactly one* tool name and one dispatch path. The load-bearing primitive is a new `ToolHandler::tool_name() -> ToolName` trait method (visible at `codex-rs/core/src/tools/handlers/mod.rs:1356` area and implemented across ~30 sites: `apply_patch.rs:248`, `list_dir.rs:581`, `mcp.rs:613`, `mcp_resource.rs:715`, `goal.rs:358,389`, `multi_agents/{spawn,wait,close_agent,resume_agent,send_input}.rs:1409,1424,1439,1454,1469`, `multi_agents_v2/{spawn,wait,list_agents,close_agent,followup_task,send_message}.rs:1496,1511,1526,1562,1577`, plus `code_mode/{execute_handler,wait_handler}.rs:109,132`).

`ToolRegistryBuilder::register_handler` now derives the registry key from `handler.tool_name()` rather than taking the name as a separate argument, which structurally eliminates the prior class of bug where a handler could be registered under one name and, internally, switch on a different invoked-name (the multi-tool dispatchers like `multi_agents.rs` and `multi_agents_v2.rs` are now split into one file per concrete tool, e.g. `multi_agents/spawn.rs` vs `multi_agents/wait.rs`).

Tests updated alongside (`codex-rs/core/src/session/tests.rs:4`, `session/tests/guardian_tests.rs:83`).

## Why it's right

- **Single-source-of-truth identity**: putting the tool name *on the handler* means there is exactly one place that defines "this code runs for that tool name." The old shape (registry-side string + handler-side switch) had two places that had to agree. A typo in one would silently route the wrong handler.
- **Splitting the multiplexers**: `multi_agents.rs:1392` becoming a directory with five concrete handlers (`spawn`, `wait`, `close_agent`, `resume_agent`, `send_input`) makes each dispatch path independently reviewable, lints catch dead match arms, and removes the "if name == 'spawn' { ... } else if name == 'wait' { ... }" pattern that was the actual home of the bug.
- **`ToolName` typed return** (rather than `&'static str`) keeps the registry index typed all the way down — a stringly-typed `tool_name()` would have re-introduced the same drift class one level deeper.
- The `code_mode/{execute,wait}_handler.rs` and `mcp_resource.rs` handlers gain the trait method along with everyone else, so the rule "every handler implements `tool_name()`" is enforced by the compiler — there is no way to ship a handler that forgets it.

## Nits

1. **No CI guard against the "two handlers, same `ToolName`" failure mode.** `register_handler` should `assert!`/return `Err` if the same `ToolName` is registered twice. With a string-keyed `HashMap` insert, the second registration silently overwrites the first today. A startup test asserting `registry.len() == expected_count` would also catch the regression.
2. **`fn tool_name(&self) -> ToolName`** takes `&self` but every implementation returns a constant. A `const TOOL_NAME: ToolName` associated constant would express the invariant ("tool name is per-type, not per-instance") and let the registry key off `T::TOOL_NAME` in generic registration paths. Acceptable as-is for trait-object dispatch, but worth a doc comment.
3. **Mass mechanical change touches 3535 lines across ~30 files** — would benefit from being split into (a) trait introduction + registry key change, (b) multi-agent multiplexer split, (c) multi-agents-v2 multiplexer split. Bisection target if a regression emerges.
4. **Sibling files in `multi_agents_v2/` (`message_tool.rs:1536` doesn't show a `tool_name()` impl in the visible diff)** — confirm every concrete handler in v2 carries the impl; if `message_tool` is a shared helper rather than a `ToolHandler`, name it accordingly so a future reader doesn't try to register it.
5. **PR description names the policy ("each registered handler instance owns exactly one tool name and one dispatch path")** but no test in the visible diff locks it. Add a `compile_fail` test or a doc-test demonstrating the invariant.

`merge-after-nits` — right architectural direction (pushing identity onto the type that owns the behavior), correctly applied to every multiplexer site, but wants the duplicate-registration guard and a smaller commit topology before merge.
