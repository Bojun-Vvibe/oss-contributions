# openai/codex PR #19771 — fix: filter dynamic deferred tools from model_visible_specs

- Link: https://github.com/openai/codex/pull/19771
- Head SHA: `f18262523e818a9fb405f53f625d2f2318813237`
- Size: +293 / -69 across multiple files

## Summary

Moves the `defer_loading` filtering of dynamic tools from `build_prompt` (called per-turn) into `ToolRouter::from_config` (called once at router construction), so `router.model_visible_specs()` is the *single* canonical source of truth for "what tools are advertised to the model". Closes the same class of bug that #19707 patched in the remote-compact path: any caller that grabbed `router.model_visible_specs()` directly (rather than going through `build_prompt`) would inadvertently expose deferred dynamic tools and trigger the `'tools.defer_loading' requires tools.tool_search` 400 from the Responses API. By baking the filter into the router itself, every existing and future caller is automatically correct.

## Specific-line citations

- `core/src/session/turn.rs:946-963`: `build_prompt` simplifies from a 17-line filter pipeline to `tools: router.model_visible_specs()`. The `filter_deferred_dynamic_tool_spec` helper and the `HashSet<ToolName>` collection of deferred names are deleted from this file. The `use codex_tools::ResponsesApiNamespaceTool` and `use codex_tools::ToolSpec` imports are correctly removed since `build_prompt` no longer constructs spec values.
- `core/src/tools/router.rs:71-93`: the inline construction of `model_visible_specs` inside `ToolRouter::from_config` now collects `deferred_dynamic_tools` from `dynamic_tools.iter().filter(|t| t.defer_loading)` and runs both the existing `code_mode_only_enabled` filter AND the new `filter_deferred_dynamic_tool_spec` in a single `filter_map` chain. The composition order is correct (code-mode first, defer-loading second) so a tool that's both nested-code-mode-hidden and deferred-dynamic is filtered exactly once.
- `core/src/tools/router.rs:296-330`: `filter_deferred_dynamic_tool_spec` is moved verbatim from `turn.rs` and gets a tasteful early-return — `if deferred_dynamic_tools.is_empty() { return Some(spec); }` — that preserves the no-deferred fast path without doing the namespace `retain` walk.
- `core/src/tools/router_tests.rs:142-225`: new `model_visible_specs_filter_deferred_dynamic_tools` integration test constructs a `ToolRouter` with one `defer_loading: true` and one `defer_loading: false` `DynamicToolSpec` under the same `codex_app` namespace, then asserts (a) `find_spec(namespaced("codex_app", hidden_tool)).is_some()` (router still knows about the deferred tool — it's only hidden from the *model-visible* projection), (b) the namespace function names returned by the unfiltered `specs()` include both, and (c) `model_visible_specs()` returns *only* the visible one. This is exactly the right shape because it pins the fundamental invariant: deferred ≠ unregistered, deferred = invisible-to-model-until-discovered.
- `core/tests/suite/compact_remote.rs:60+`: new `assert_tools_payload_does_not_defer` helper recursively walks the wire-level `tools` JSON for any `defer_loading` field and fails if found — this is an integration-level pin against regression at the HTTP-payload boundary, not just the Rust API boundary, which is the right level to assert at given the bug class is "wire-format-rejected-by-server".

## Verdict

**merge-as-is**

## Rationale

This is the correct architectural fix: the per-call-site filter pattern was already proven to leak (#19707 had to apply it specifically to the remote-compact path), and the next caller to grab `model_visible_specs()` directly would have hit the same 400. Pushing the filter into the router constructor makes the API actually mean what its name says — `model_visible_specs()` returns what's visible to the model, full stop, no caller-side post-processing required.

The router-level test pins the API contract precisely (specs() vs model_visible_specs() asymmetry, find_spec() still works for execution-time dispatch of the deferred tool once it's discovered), and the `compact_remote.rs` wire-payload assertion is the right belt-and-suspenders pin. The fast-path `if deferred_dynamic_tools.is_empty()` early-return at `:301-303` keeps the hot path zero-cost for the common case of no deferred tools. No regressions visible: the previously-deleted `filter_deferred_dynamic_tool_spec` body in `turn.rs` is reproduced verbatim in `router.rs` so semantics are pinned-equal across the move.

