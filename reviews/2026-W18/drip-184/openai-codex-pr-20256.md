---
pr: openai/codex#20256
sha: dcb344505c72cba91917185af9e67b028075d1ba
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# tools: namespace tool_suggest

URL: https://github.com/openai/codex/pull/20256
Files: `codex-rs/tools/src/tool_discovery.rs`, `codex-rs/tools/src/lib.rs`,
`codex-rs/tools/src/tool_discovery_tests.rs`,
`codex-rs/tools/src/tool_registry_plan.rs`,
`codex-rs/tools/src/tool_registry_plan_tests.rs`,
`codex-rs/core/src/tools/spec_tests.rs`,
`codex-rs/core/tests/suite/tool_suggest.rs`
Diff: 99+/94-

## Context

`tool_suggest` is a built-in local tool (suggests connectors / discoverable
plugins to install when no current tool fits). Until this PR it was
emitted as a top-level Responses-API function, which planted it in the
default function namespace alongside user-installed tools — inconsistent
with the project's own convention that built-in local tools (apps,
plugins, etc.) live under their own namespace so the model can reason
about provenance and the registry can group/permission them by namespace.
The fix migrates `tool_suggest` to a `Namespace` ToolSpec while
preserving the wire-level callable so existing callers don't break.

## What's good

- Constant split at `tool_discovery.rs:18-19` is the load-bearing rename:
  `TOOL_SUGGEST_TOOL_NAMESPACE = "tool_suggest"` (the namespace label
  emitted to Responses) and `TOOL_SUGGEST_TOOL_NAME = "tool_suggest_tool"`
  (the function name *inside* the namespace). The old constant kept its
  name but its value changed from `"tool_suggest"` to `"tool_suggest_tool"`
  — the reused-symbol rename is the right call because every grep'able
  call site now has to be updated, surfacing latent forks via the
  compiler.
- Re-export at `lib.rs:115` adds `TOOL_SUGGEST_TOOL_NAMESPACE` next to
  the existing `TOOL_SUGGEST_TOOL_NAME` so external crates have both
  identifiers without reaching into `tool_discovery::*`.
- `create_tool_suggest_tool` at `tool_discovery.rs:312-336` swaps
  `ToolSpec::Function(...)` for `ToolSpec::Namespace(ResponsesApiNamespace
  { name: NAMESPACE, description: default_namespace_description(NAMESPACE),
  tools: vec![ResponsesApiNamespaceTool::Function(...)] })`. Single
  function inside a namespace is the right shape for a one-tool built-in
  (matches how other one-call namespaces are emitted) and uses the shared
  `default_namespace_description` helper rather than rolling a custom
  string — keeps surface consistent with the rest of the namespace
  catalog.
- The embedded prompt template in the function description at
  `tool_discovery.rs:312` correctly references `{TOOL_SUGGEST_TOOL_NAME}`
  in the "call `tool_suggest_tool` with…" instruction (verifiable in the
  fixture diff at `tool_discovery_tests.rs:140` where the snapshot now
  reads `If one tool clearly fits, call \`tool_suggest_tool\` with`) —
  so the *model-facing* call instruction tracks the new function name,
  not the namespace.
- Test plumbing at `tool_suggest.rs:13-15,29-30` removes the duplicated
  local `const TOOL_SEARCH_TOOL_NAME` and `const TOOL_SUGGEST_TOOL_NAME`
  in favour of the canonical exports, and removes the now-unused
  `function_tool_description` helper in favour of
  `namespace_child_tool(body, NAMESPACE, NAME)` which descends into the
  emitted namespace structure instead of pretending the tool is still a
  top-level function — exactly the shape change the test surface should
  enforce.
- Per-feature gate test at `spec_tests.rs:858` was updated to assert
  absence by `TOOL_SUGGEST_TOOL_NAMESPACE` (the visible top-level name in
  the Responses tools array) rather than by the now-internal function
  name — correct because the namespace label is what `tool.name()`
  returns at the top level.

## Nits

- The constant rename trick — same Rust identifier
  (`TOOL_SUGGEST_TOOL_NAME`) now points at a different string value
  (`"tool_suggest_tool"`) — is friendly to the compiler but hostile to
  anyone reading a stack trace, log line, or analytics event from a
  prior build that mentions `"tool_suggest"`. Would be safer to retire
  the old identifier entirely (`TOOL_SUGGEST_FUNCTION_NAME` for the new
  one) so future grep on `tool_suggest` resolves unambiguously to the
  namespace.
- `default_namespace_description(NAMESPACE)` at `tool_discovery.rs:314`
  produces the generic `"Tools in the {namespace} namespace."` blurb
  visible in the fixture diff. For a one-tool namespace whose function
  has a 60-line embedded prompt, the namespace-level description is
  pure noise to the model. Worth either (a) special-casing single-tool
  namespaces to surface the inner function description at namespace
  level too, or (b) at minimum a one-line "Suggests installing
  connectors or plugins not currently in the active tools list." string
  so a model reading the namespace index gets a discriminator.
- No test asserting backwards-compat on the *external* wire name —
  past sessions/transcripts that recorded a tool call to
  `"tool_suggest"` (old top-level name) will now fail to dispatch
  because the model must call `tool_suggest.tool_suggest_tool`. If the
  intent is "session replay still works", the PR needs an alias /
  legacy dispatcher; if not, it needs a CHANGELOG / release-notes entry
  flagging the rename for any external automation that pinned to the
  old function name.
- `tool_registry_plan.rs` / `tool_registry_plan_tests.rs` are touched
  in the same commit; quick audit of registry-plan ordering would
  confirm namespace-vs-function emission order is stable across
  features (the model relies on this for prompt cache hits).

## Verdict reasoning

Mechanical, locally-correct migration of a built-in tool from top-level
function to its rightful namespace, with the test surface honestly
updated to assert the new shape (not the prior shape with a renamed
constant). Pulls the spec_tests/tool_suggest.rs assertions onto the
canonical exports so the whole codebase agrees on what the tool is
called now. Nits are about the grep-friendliness of the constant rename
trick, fattening up the boilerplate namespace description, and a
one-line release note for anyone who pinned to `"tool_suggest"` at the
wire level — none blocking, but the wire-rename note is worth catching
before merge.
