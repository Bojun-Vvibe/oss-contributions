# openai/codex #21231 — Support Always Allow for MCP app messages

- Author: leoshimo-oai
- Head SHA: `9f74246ee0762119734a1502f6167fca95249f24`

## Files changed (top)

- `codex-rs/config/src/mcp_types.rs` (+11 / -0) — adds `McpAppMessageApprovalMode` enum
- `codex-rs/app-server-protocol/src/protocol/v2.rs` (+9 / -0)
- `codex-rs/config/src/mcp_edit.rs` (+7 / -0) — TOML serialization
- `codex-rs/app-server/src/config_manager_service_tests.rs` (+46 / -0) — round-trip test
- Auto-generated schema/TS bindings across multiple files
- Plus a sweep through `*_tests.rs` adding the new field to existing struct literals

## Observations

- `codex-rs/config/src/mcp_types.rs:25-31` — `McpAppMessageApprovalMode { Prompt, Approve }` is a clean two-variant enum, semantically parallel to the existing `AppToolApproval`. Reasonable to add as a separate type rather than reusing `AppToolApproval` because app-message approval has its own semantics (granting a tool-execution-approval ≠ granting an app-message-display-approval). Naming choice good.
- `codex-rs/config/src/mcp_types.rs:65-67` — new `mcp_app_message_approval_mode: Option<McpAppMessageApprovalMode>` on `McpServerToolConfig` with `#[serde(default, skip_serializing_if = "Option::is_none")]` is the correct shape — `None` means "inherit/use default", and won't pollute serialized TOML for users who never set it.
- `codex-rs/config/src/mcp_edit.rs:236-241` — manual TOML emission mirrors the existing `approval_mode` block immediately above it; symmetry preserved. Good.
- `codex-rs/app-server/src/config_manager_service_tests.rs:237-280` — full write-then-read round-trip with `MergeStrategy::Upsert`, asserting the value lands at `mcp_servers.docs.tools.search.mcp_app_message_approval_mode = "approve"`. This is the right shape of test for a config-plumbing PR.
- `codex-rs/config/src/mcp_types_tests.rs:323` and serialization assert at `:341` — round-trip coverage at the type level too. Symmetric.
- The PR title is "Support Always Allow for MCP app messages" but the actual *enforcement* of the approval mode (i.e., where in the runtime path the new field is *consumed* to skip a prompt) is not in this diff — only the schema/storage plumbing is. Search of the diff for `mcp_app_message_approval_mode` shows no read-side consumer. This is fine if it's part 1 of a stack, but worth confirming the consumer PR exists or is queued; otherwise the field lands dead.
- All 12+ `*_tests.rs` files updated mechanically with `mcp_app_message_approval_mode: None` added to existing `McpServerToolConfig {…}` literals — clean, no missed sites visible. Generated schema files (`*.schemas.json`, `*.ts`) are consistent.

## Verdict

`merge-as-is`

Pure plumbing PR — new config field, serialization, schema regen, test. No behavior change in the runtime path yet. The patterns mirror the immediately-adjacent `approval_mode` precedent precisely. Only "concern" is the dead-field risk if the runtime-consumer follow-up doesn't land soon, which is a sequencing question for the maintainer rather than a code defect.
