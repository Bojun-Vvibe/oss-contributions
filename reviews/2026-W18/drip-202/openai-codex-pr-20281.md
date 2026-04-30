# openai/codex #20281 — Use selected turn environments for runtime context

- **Author:** starr-openai
- **SHA:** `21fea44`
- **State:** OPEN
- **Size:** +483 / -118 across `core/src/environment_selection.rs`, `core/src/session/{mod,mcp,review,session}.rs`, `codex-mcp/src/runtime.rs`, plus tests
- **Verdict:** `merge-after-nits`

## Summary

Promotes `TurnEnvironmentSelection` from a passive selection list into the
runtime source of truth for `cwd` and `McpRuntimeEnvironment` resolution.
Three concrete additions:

1. **Duplicate-id guard** at `environment_selection.rs:29-37` —
   `validate_environment_selections` now refuses two `TurnEnvironmentSelection`s
   with the same `environment_id` (was silently ambiguous before; "first wins"
   was implicit), with a regression at `:128-148`.
2. **`primary_selected_cwd_or_fallback`** at `environment_selection.rs:70-78`
   threads the first selection's `cwd` into `Codex::spawn`'s plugin-loader and
   `AgentsMdManager` paths at `session/mod.rs:474-501`, replacing the prior
   "always use `config.cwd`" behavior. Locked by two tests at `:152-172`.
3. **`mcp_runtime_environment_for_turn_context` /
   `_for_configuration`** at `session/mcp.rs:3-46` centralize the
   primary-environment-or-local fallback so MCP turn dispatch and stored
   session-config-restore go through one definition each.

## Reasoning

The refactor is the right shape — the prior code spread "use the selected
environment if any, else fall back to local" across at least three call sites
with subtly different fallbacks (`turn_context.environment.unwrap_or_else(local)`
in `mcp.rs:279-285` and an implicit `config.cwd` everywhere else). Centralizing
into named helpers + adding `validate_environment_selections`'s duplicate
guard closes the "two selections with same id, second's cwd silently shadows
first" footgun. The duplicate-rejection regression at `:128-148` correctly
tests both that the error fires and that the message contains the literal
"duplicate" string.

Two pre-merge concerns:

1. **`primary_selected_cwd_or_fallback` is `pub(crate)`** but it's now
   load-bearing for plugin discovery (`plugins_for_config(&load_config)` at
   `session/mod.rs:481`) and skill discovery (`skills_load_input_from_config(&load_config, ...)` at
   `:483`) — both of which previously used `config.cwd` directly. The
   semantic shift is large enough to warrant a doc-comment on the helper
   spelling out: "callers that historically passed `config.cwd` to a
   resource-discovery API must now pass `primary_selected_cwd_or_fallback(&environments,
   &config.cwd)` instead, otherwise resource discovery happens against the
   wrong working directory when the user has selected a non-local environment."

2. **`mcp_runtime_environment_for_configuration` at `session/mcp.rs:18-46`
   logs a warning and silently falls back to the local environment when a
   stored `selected_environment.environment_id` is unknown** ("`unknown
   stored MCP environment id `{}`; falling back to local environment`"). For
   a user who saved a session against a remote sandbox that has since been
   torn down, this means MCP turn dispatch silently runs against `local`
   with no UI surface. A `PermissionsInstructions` warning into the
   transcript or a `task_event` of kind `environment_unavailable` would
   make the fallback observable rather than silent.

Smaller nits: the `pub(crate) fn environment` / `pub fn environment`
visibility flip at `codex-mcp/src/runtime.rs:53,57` makes the helpers part of
the public crate API without a doc-comment explaining the contract; the
`session/review.rs:126` removal of `environment: parent_turn_context.environment.clone()`
relies on the `environments` Vec being populated by then — confirm there's
no test path that constructs a review thread before turn-environments have
been validated.
