# PR #19836 — disallow fileparams metadata for custom mcps

- **Repo**: openai/codex
- **PR**: #19836
- **Head SHA**: `3bdb43a0`
- **Author**: colby-oai
- **Size**: +31 / -3 across 2 files
- **Verdict**: **merge-as-is**

## Summary

Tightens `lookup_mcp_tool_metadata` so that the `openai/fileParams`
meta key (which controls which MCP tool input parameters are
treated as file uploads on the OpenAI/Responses-API side) is
honored *only* when the MCP server is the trusted built-in
`CODEX_APPS_MCP_SERVER_NAME`. For every other server (i.e. all
user-installed custom MCPs), `openai_file_input_params` is now
forced to `None` regardless of what the server's tool meta block
declares. This closes a quiet privilege-escalation shape: a
malicious or sloppy custom MCP could otherwise advertise
`openai/fileParams: ["arbitrary_param"]` and have codex
silently treat that parameter as a file-upload field, which
changes how the value is forwarded upstream.

## Specific changes

- `codex-rs/core/src/mcp_tool_call.rs:1197-1213` — replaces the unconditional `Some(declared_openai_file_input_param_names(tool_info.tool.meta.as_deref())).filter(|params| !params.is_empty())` call with a delegated `openai_file_input_params_for_server(server, tool_info.tool.meta.as_deref())` helper that early-returns `None` if `server != CODEX_APPS_MCP_SERVER_NAME`. The helper is `pub(crate)` only — correct visibility, doesn't widen the public surface.
- `codex-rs/core/src/mcp_tool_call_tests.rs:184-201` — new unit test `openai_file_params_are_only_honored_for_codex_apps` that constructs a meta block declaring `openai/fileParams: ["file"]` and asserts the helper returns `Some(vec!["file"])` for `CODEX_APPS_MCP_SERVER_NAME` and `None` for `"minimaltest"`. Pins both sides of the gate at exactly the boundary the new code introduces, so a future refactor that loses the server check fails loudly.

## Risks

- **Silent capability loss for any external MCP that today relies on `openai/fileParams`**: if a third-party MCP author shipped a tool that expected file-upload semantics on a named param, that contract is now silently broken. Probably zero deployed users (the meta key looks bespoke to the codex-apps integration), but a release-notes line saying "if you maintain a custom MCP that uses `openai/fileParams`, that key is now ignored — use the codex-apps MCP if you need file-upload semantics" would close the surprise gap.
- **Server allow-list is exact-string equality**: if codex ever introduces a second trusted server (e.g. `CODEX_APPS_INTERNAL_MCP_SERVER_NAME` for a staging variant), the gate at `:1207` becomes a maintenance hazard. A `const TRUSTED_FILE_PARAM_SERVERS: &[&str] = &[...]` with a `.contains()` would scale better, but for the current single-server world the inline equality check is the simplest right answer.
- **No test for the negative path inside the full `lookup_mcp_tool_metadata` flow**: the test calls `openai_file_input_params_for_server` directly, which proves the helper is correct in isolation but doesn't prove that `lookup_mcp_tool_metadata` actually wires the helper into the result struct under both branches. A fuller test that builds a `tool_info` for a non-codex-apps server and asserts the resulting `McpToolMetadata.openai_file_input_params == None` would cover the integration boundary.

## Verdict

`merge-as-is` — small, surgical, and exactly the right shape: a
positive allow-list at the lowest layer that handles file-param
metadata, with a unit test pinning both sides of the gate. The
helper is correctly `pub(crate)` and the test calls it directly
which is the right surface for a security-property test
(integration tests would obscure which layer actually enforces
the rule). The diagnosis ("custom MCPs should not be able to
opt themselves into the file-upload param contract") is the
kind of capability-narrowing change that gets cheaper the
earlier it lands, so blocking on the nice-to-have integration
test would be net-negative.

## What I learned

The classic shape of this bug is "trust attribute lives on the
data instead of on the relationship". The original code asked
"does this tool's meta declare `openai/fileParams`?" — the data
itself decides. The fix asks "does this tool's *server* belong
to the set we trust to declare `openai/fileParams`?" — the
relationship between the consumer (codex) and the producer (the
MCP server) decides. The same shape shows up everywhere MCP /
plugin / extension surfaces let third parties annotate their
own capabilities. Defaults must be `None` and capability grants
must be allow-listed by identity, never by self-declaration in
the metadata of the thing being granted.
