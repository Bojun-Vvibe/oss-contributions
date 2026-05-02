---
pr: https://github.com/openai/codex/pull/20703
head_sha: 365bf45526d6441d074dc05a877c5ae8119e5907
author: abhinav-oai
additions: 562
deletions: 33
files_touched: 10
---

# Scope

Adds support for the `updatedToolOutput` / `updatedMCPToolOutput`
hook-output fields in `PostToolUse` so hook authors can replace the
model-visible tool result without altering the executed-command result
or telemetry. `updatedMCPToolOutput` is ignored when `updatedToolOutput`
is set. Touches the tool registry plumbing, the dispatch trace, schema,
hook output parser, and adds two integration test files. Author flagged
this PR's own header as "Halt — Maybe de-prio? Very very few mentions
of either" by way of explicit GitHub-search links — i.e. usage of these
hook fields outside reference docs is essentially zero today.

# Specific findings

- `codex-rs/core/src/tools/registry.rs:111` (head `365bf45`) — new
  `model_visible_override: Option<FunctionToolOutput>` field on
  `AnyToolResult`. The dual-render path in
  `AnyToolResult::into_response()` (registry.rs:118-128) chooses the
  override for the model-facing response item but `code_mode_result()`
  is intentionally *not* overridden, so `ResponseInputItem::Code Mode` paths
  still see the original. The new test
  `model_visible_override_does_not_replace_typed_tool_output`
  (`registry_tests.rs:60-104`) pins this contract — good. Document the
  asymmetry in a doc-comment on the field; future maintainers will
  otherwise assume "override = override everywhere".
- `codex-rs/core/src/tools/registry.rs:487-494` — when the hook supplies
  `updated_tool_output`, the value is wrapped via
  `FunctionToolOutput::from_text(post_tool_use_output_to_model_text(updated_tool_output), Some(true))`.
  The `Some(true)` `success` flag is a silent assumption: the original
  tool may have failed, but the override is unconditionally marked
  successful. If a hook author redacts a *failed* output's text without
  intending to flip its success bit, downstream model behavior will drift.
  Suggest preserving the original output's success flag (or accept a
  hook-supplied success override).
- `codex-rs/core/src/tools/registry.rs:534-545` —
  `post_tool_use_output_to_model_text` collapses non-string JSON values
  via `Value::to_string()`. For a structured replacement like
  `{"redacted": "..."}` the model sees the literal JSON. That's fine for
  text-only function outputs, but the doc-comment should call out
  explicitly that hook authors targeting MCP tools should use
  `updatedMCPToolOutput` if they want structured passthrough — otherwise
  a hook setting `updatedToolOutput` on an MCP call will downgrade
  structured content to a JSON string.
- `codex-rs/core/src/tools/tool_dispatch_trace.rs:39-100` — the trace
  helper threads the override through
  `tool_dispatch_result(...)` so the recorded dispatch artifact reflects
  what the model actually saw. Good for debuggability; no concerns.
- `codex-rs/core/tests/suite/hooks.rs:462-475` — fixture script gains
  `updated_tool_output` and `invalid_updated_tool_output` modes. The
  `invalid_*` mode constructs `{"redacted": reason}` — i.e. a structured
  value where the original `tool_response` was a string — and the test
  `post_tool_use_ignores_updated_tool_output_with_wrong_builtin_kind`
  (hooks.rs:2829-2880) asserts the original output survives. The
  validation rule is "by JSON kind against the original hook-facing
  tool_response", per the PR body; that rule isn't visible in the diff
  excerpt I read — confirm it lives in the hook output parser
  (`hooks/src/events/post_tool_use.rs`, +194 lines).
- `codex-rs/hooks/src/engine/output_parser.rs` (+14/-17) — small diff,
  presumably wires `updatedToolOutput` through the parser. Worth a
  second pair of eyes on the parser's behavior when both
  `updatedToolOutput` and `updatedMCPToolOutput` are present (PR body
  states the former wins; pin with a unit test if not already).

# Suggested changes

1. Preserve the original tool result's `success` flag when wrapping the
   `updatedToolOutput` replacement, instead of unconditional
   `Some(true)`.
2. Add a doc-comment on `AnyToolResult::model_visible_override`
   explaining that `code_mode_result()` is not overridden.
3. Add an explicit unit test for the precedence rule
   `updatedToolOutput > updatedMCPToolOutput` when both are supplied.
4. Author's "de-prio" caveat is fair — it might be worth gating this
   behind a feature flag until usage materializes, to keep the hook
   surface area honest.

# Verdict

`merge-after-nits`

# Confidence

Medium. The plumbing is clean and the new tests are substantive; my
concerns are local correctness questions (success flag, structured/MCP
asymmetry) rather than architectural ones.

# Time spent

~12 minutes.
