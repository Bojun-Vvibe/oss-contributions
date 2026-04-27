# openai/codex PR #19707 — Fix remote compaction with deferred dynamic tools

- Link: https://github.com/openai/codex/pull/19707
- Head SHA: `437f3d131678efd452112e3d90692ecd382edac0`
- Size: +126 / -16 across 2 files

## Summary

Fixes a 400 from the remote compact endpoint — `Invalid Value: 'tools.defer_loading'. Deferred tools require tools.tool_search.` — that surfaced after tool discovery started enabling deferred dynamic tools more aggressively. Root cause: `compact_remote.rs` was building its `Prompt.tools` from the raw `tool_router.model_visible_specs()`, while normal-turn `build_prompt` already filtered out tool specs marked `defer_loading=true` before sending. When the deferred tool's paired top-level `tool_search` wasn't preserved into the compact path, the API correctly rejected the request. The fix extracts the existing filter into a reusable `model_visible_specs_for_prompt(router, turn_context)` helper in `session/turn.rs` and uses it in both call sites.

## Specific-line citations

- `compact_remote.rs:15` adds `use crate::session::turn::model_visible_specs_for_prompt;` — single new import, no other coupling.
- `compact_remote.rs:165-168` is the one-line behavior change: `tools: tool_router.model_visible_specs()` → `tools: model_visible_specs_for_prompt(tool_router.as_ref(), turn_context.as_ref())`. Same shape as the normal-turn build, no other compact-prompt fields touched.
- `session/turn.rs:945-958` is the refactor of `build_prompt`: 14 lines of inline deferred-tool filtering (HashSet construction + filter_map) collapse to `let tools = model_visible_specs_for_prompt(router, turn_context);` — pure extract-method, the resulting `tools` value is byte-identical.
- `session/turn.rs:963-985` is the new shared helper pair: `pub(crate) fn model_visible_specs_for_prompt(...) -> Vec<ToolSpec>` is the public seam, `fn filter_deferred_dynamic_tool_specs(specs, dynamic_tools)` is the private implementation. Splitting the helper into "takes a `TurnContext`" public surface and "takes a slice" inner implementation is the right shape because it keeps the public contract narrow (`turn_context.dynamic_tools.as_slice()` is the only field consumed) and makes the inner function unit-testable without constructing a full `TurnContext`.
- The `if deferred_dynamic_tools.is_empty()` early-return at `:980` preserves the original fast-path (no per-spec `filter_map` cost when nothing is deferred) — important because `build_prompt` runs every turn.

## Verdict

**merge-as-is**

## Rationale

This is the textbook way to fix a "two paths got out of sync" bug: extract the shared logic into a function called from both sites, prove byte-identity via the original code's behavior at the empty-deferred-set fast-path (the original `if deferred_dynamic_tools.is_empty()` branch is preserved verbatim in the new helper), and verify against the real endpoint. The PR body explicitly says this was smoke-tested against the live compact endpoint with the actual `codex_app::automation_update` dynamic tool from `openai/openai#849632`, with the failure reproduced first (raw tool list returns 400 with the named error string) and the fix verified (filtered list returns 200 with `compaction_summary`). That's the right kind of evidence for an API-contract fix.

The pattern of "deferred dynamic tools must coexist with `tool_search`" is now expressible in exactly one place, so future call sites (e.g. local compaction, prompt-debug dumps, conversation-export tools) inherit the correct filtering by importing the helper instead of re-implementing the HashSet+filter_map dance. Worth adding a unit test on `filter_deferred_dynamic_tool_specs` directly that asserts the `tool_search` spec passes through and the `defer_loading: true` specs are dropped — the contract is currently only validated by an integration smoke test, and a unit-level pin would catch a regression on the helper itself if a future refactor inlines it differently.
