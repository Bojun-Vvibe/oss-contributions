# openai/codex #20750 — Unify skip-review handling for approval_mode = "approve"

- **Repo:** openai/codex
- **PR:** #20750
- **URL:** https://github.com/openai/codex/pull/20750
- **Head SHA:** `a2c2892d06e5b026641b7d81378eaa7386a899f4`
- **Files touched:** 34 files across `codex-rs/{cli,codex-mcp,config,core,core-plugins}`
  (+~470 / -~250 by file count). The substantive logic lives in
  `codex-rs/core/src/mcp_tool_call.rs` (+83 -24) and the test churn in
  `mcp_tool_call_tests.rs` (+126 -56). Most other files are
  one-line additions of a new `default_tools_approval_mode: None`
  field threaded through fixtures.
- **Verdict:** needs-discussion

## Summary

This PR replaces the `server == CODEX_APPS_MCP_SERVER_NAME` string
comparisons in the MCP tool-call dispatcher with a runtime
`is_host_owned_codex_apps_server(&server)` check on the connection
manager, then plumbs the resulting boolean through
`handle_mcp_tool_call`, `handle_approved_mcp_tool_call`,
`maybe_track_codex_app_used`, and `build_mcp_tool_call_request_meta`.
The motivation (per title) is to unify how `approval_mode = "approve"`
is detected for the host-owned Codex Apps server vs. user-installed MCP
servers that happen to share the well-known name.

## Specific notes

- **`codex-rs/core/src/mcp_tool_call.rs:122-126`** — the new
  `is_host_owned_codex_apps_server` lookup acquires a `read()` on
  `mcp_connection_manager` once at the top of `handle_mcp_tool_call`
  and again at line 313 in `handle_approved_mcp_tool_call`. Two
  separate read locks around a TOCTOU window: between dispatch and
  approval, a re-init of the connection manager could in principle
  flip the flag. Probably fine in practice (re-init is rare), but
  worth either (a) caching the result on the `Invocation`/`TurnContext`
  or (b) documenting the assumption.
- **`build_mcp_tool_call_request_meta` (line 845, signature change)** —
  the function now takes `is_host_owned_codex_apps_server: bool`
  instead of `server: &str`. Caller at line 182 is updated, but
  semantically this couples the meta-builder to the dispatcher's
  classification. If any other site ever wants to call this builder it
  has to redo the classification. Consider keeping the `server: &str`
  parameter and doing the classification inside.
- **Test fixture churn** — ~20 of the 34 files are pure
  `default_tools_approval_mode: None` additions. The new field is
  added to `AppToolApproval`/related config in
  `codex-rs/config/src/mcp_types.rs:+13`, but I don't see it actually
  *consumed* in this diff outside of plumbing. Either the consumer
  lives behind a feature gate I missed, or this is dead config. Worth
  the author confirming.
- **`mod_tests.rs:430-489`** — the `mcp_prompt_auto_approval_honors_auto_review_approved_tools`
  test was restructured to wrap the approve-mode assertions in a loop
  rather than inline them; the new `tool_approval_mode: Some(AppToolApproval::Auto)`
  case at line 489 is genuinely new coverage. Good.

## Verdict rationale

The refactor is in the right direction (string-name comparisons for
identity are a fragility) but the diff is large and introduces a new
config field whose consumer isn't visible in this PR. Recommend
splitting into (1) `is_host_owned_codex_apps_server` plumbing only and
(2) the `default_tools_approval_mode` field + its consumer in a
follow-up. As-is it's hard to review the unify-the-skip-review claim
end-to-end.
