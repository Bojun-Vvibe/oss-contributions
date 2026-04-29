# PR #20091 — Move tool_suggest into its own namespace

- **Repo:** openai/codex
- **Link:** https://github.com/openai/codex/pull/20091
- **Author:** mzeng-openai
- **State:** DRAFT
- **Head SHA:** `50f4f6be168a5ba2a2ef8ec65283ef72bbd93d97`
- **Files:** `codex-rs/core/src/tools/router.rs`, `codex-rs/core/src/tools/router_tests.rs`, `codex-rs/core/tests/suite/tool_suggest.rs`, `codex-rs/tools/src/lib.rs`, `codex-rs/tools/src/tool_discovery.rs`, `codex-rs/tools/src/tool_discovery_tests.rs` (+169/-79)

## Context

`tool_suggest` was previously registered as a flat `ToolSpec::Function`. That worked but conflated two roles: it's both an *operation* the model invokes and a *capability surface* the model navigates. The Responses API namespace primitive (`ToolSpec::Namespace`) exists precisely for this — a parent grouping with a description plus child function-tools that share the namespace's identity. This PR converts `tool_suggest` to that shape so future "siblings" (e.g. a `tool_suggest.rate` or `tool_suggest.dismiss`) can join the namespace without churning the registration sites.

## What changed

1. **`tools/src/tool_discovery.rs:18` + `lib.rs:114`** — adds `pub const TOOL_SUGGEST_NAMESPACE_NAME: &str = "tool_suggest"` next to the existing `TOOL_SUGGEST_TOOL_NAME` (same string today, but distinct conceptually). Re-exports both from the crate root.

2. **`create_tool_suggest_tool` rewrite at `tool_discovery.rs:315-336`** — replaces the old `ToolSpec::Function(ResponsesApiTool { name: TOOL_SUGGEST_TOOL_NAME, ... })` with `ToolSpec::Namespace(ResponsesApiNamespace { name: TOOL_SUGGEST_NAMESPACE_NAME, description: "Suggest missing connectors or plugins ...", tools: vec![ResponsesApiNamespaceTool::Function(ResponsesApiTool { name: TOOL_SUGGEST_TOOL_NAME, ... })] })`. The namespace gets a short description; the heavy multi-paragraph workflow description stays on the inner function-tool.

3. **`tools/router.rs:144-167` — parallel-call lookup** — this is the substantive router change and the part most worth reviewing. The old `configured_tool_supports_parallel` had:

   ```rust
   if tool_name.namespace.is_some() { return false; }
   ```

   That was a hard cutoff: any namespaced call defaulted to "no parallel support." The PR removes the early-return and instead extends the match arm:

   ```rust
   ToolSpec::Namespace(namespace) => namespace.tools.iter().any(|tool| match tool {
       ResponsesApiNamespaceTool::Function(tool) => {
           tool_name.namespace.as_deref() == Some(namespace.name.as_str())
               && tool.name == tool_name.name
       }
   }),
   ```

   So a call to `(namespace=tool_suggest, name=tool_suggest)` against a registered namespace whose `supports_parallel_tool_calls = true` correctly resolves to `true`. The `Function`/`Freeform` arms gain `if tool_name.namespace.is_none()` guards so a namespaced call doesn't accidentally match a flat-registered function of the same leaf name.

4. **`router_tests.rs:148-188`** — the directly-targeted test: a router with `tool_suggest` registered as a namespace asserts `tool_supports_parallel(ToolName::namespaced("tool_suggest","tool_suggest")) == true` *and* `tool_supports_parallel(ToolName::plain("tool_suggest")) == false`. The negative half is what makes the test trustworthy — without it you couldn't tell the new code from a "return true always" bug.

5. **`tests/suite/tool_suggest.rs:48-77` — `function_tool_description`** — the helper that walked the wire-shape JSON to pull a tool's description had to learn the new shape. Old: looked at top-level `tools[]` for `name`+`description`. New: detects `type == "namespace"`, descends into `tools[]` of the namespace, finds the matching child by name, returns its description. The wire shape this is reading is what actually goes to the model.

## Design analysis

Three observations:

1. **The `configured_tool_supports_parallel` rewrite is the load-bearing change.** The parent-PR re-shaping is mostly mechanical; this router-side change is the part where a refactor could silently lose a capability. The new match arms are correct: `Function`/`Freeform` only match plain calls; `Namespace` only matches when the call's `namespace` matches the namespace's `name` *and* the call's `name` matches one of the namespace's child function-tools. The `&& tool.name == tool_name.name` and the `tool_name.namespace.as_deref() == Some(namespace.name.as_str())` are both essential — drop either and you get false positives.

2. **`namespace.tools.iter().any(...)` short-circuits correctly.** For a namespace with one child (today's case), this is one comparison. For a future N-child namespace, it's O(N) per call but `N` will be tiny.

3. **Why both `TOOL_SUGGEST_NAMESPACE_NAME` and `TOOL_SUGGEST_TOOL_NAME` are the same string.** Today they happen to both be `"tool_suggest"`. Keeping them as separate constants is the right call — when the namespace grows a second child (e.g. `dismiss`), only the leaf-tool constant changes per registration. Worth a one-line comment at the constant pair clarifying "same string today, may diverge."

## Risks

1. **MCP-server-side parallel arm.** `configured_tool_supports_parallel` is one of two paths in this file (the other is `mcp_parallel_support_uses_exact_payload_server` per the existing test name). The PR's diff doesn't touch the MCP-side parallel logic. If MCP tools are also moving to namespaces in a follow-up, that helper will need the symmetric fix. Worth a comment in the PR body noting the MCP-side is intentionally untouched in this scope.

2. **Wire-format compatibility for older models.** `ToolSpec::Namespace` serializes to `{type: "namespace", tools: [...]}`. If any code path serializes the tool list for a model variant that doesn't understand `type: "namespace"`, the `tool_suggest` capability would silently disappear from that model's view. The `function_tool_description` helper rewrite at `tool_suggest.rs:48-77` proves at least one wire-reading path knows the new shape; worth grep-confirming there's no other JSON-walker that still expects flat function tools at the top level.

3. **Backward-compat for in-flight tool_calls.** If a model previously emitted `{"type":"function","name":"tool_suggest"}` (no namespace) and the rollout/replay machinery tries to route it through the new namespaced router, the `Function` arm of the parallel-support check now requires `tool_name.namespace.is_none()` — that part still matches plain calls — but the *dispatch* path needs the same dual-shape tolerance. Worth confirming `ToolRouter::dispatch` (or whatever the runtime call path is) accepts both `(plain, "tool_suggest")` and `(namespaced, "tool_suggest", "tool_suggest")` for at least one release window.

4. **Draft state.** PR is marked DRAFT. Don't merge until the author flips it to ready and confirms the missing wire-back-compat / MCP-side answers above.

## Verdict

**Verdict:** needs-discussion

Refactor shape is right and the parallel-support helper is correctly generalized rather than special-cased. But three open questions block merge:
- Wire-format compatibility for in-flight `(plain, "tool_suggest")` calls during rollout — confirm dispatch accepts both shapes.
- MCP-side parallel-support helper untouched — intentional? Document the scope.
- Draft status — author signals more work pending.

The router-test pair (positive + negative cell) is exactly the right test shape; ship that test pattern even if the namespace move ends up reverted.

---

*Reviewed by drip-156.*
