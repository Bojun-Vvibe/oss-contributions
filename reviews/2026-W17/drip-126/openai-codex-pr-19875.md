# openai/codex #19875 — Strip connector provenance metadata from custom MCP tools

- URL: https://github.com/openai/codex/pull/19875
- Head SHA: `8cf1249cd6e3e9671c659df4e47da476c3c6c173`
- Verdict: **merge-as-is**

## Review

- Real security fix at `codex-rs/codex-mcp/src/rmcp_client.rs:369-392`: untrusted MCP servers (anything other than the first-party `CODEX_APPS_MCP_SERVER_NAME`) can no longer forge `connector_*` metadata that would let codex's internal callable-name normalization classify their tools as belonging to a trusted connector (Gmail, GDrive, etc.). `sanitize_tool_connector_metadata` is the single-purpose gate: trusted server → pass through; everything else → `strip_untrusted_connector_meta(tool)` and force the returned `(connector_id, connector_name, connector_description)` triple to all-`None`.
- Stripping is *case-insensitive* prefix-match at `:385-388` (`prefix.eq_ignore_ascii_case(CONNECTOR_META_PREFIX)` with `CONNECTOR_META_PREFIX = "connector"`), which correctly catches the four trick-key shapes a malicious server might try: `connector_id`, `connectorDescription`, `CONNECTOR_UPPERCASE`, `connectorFutureField`. Test case at `:439-447` enumerates exactly these — the test is the spec for the policy, which is the right way to pin a security invariant.
- Allowlist of preserved keys at the same test (`openai/fileParams`, `custom`) confirms the scope is narrow: only the `connector*` namespace is stripped, all other tool metadata flows through untouched. This means upstream additions (`openai/*`, `mcp/*`, vendor namespaces) keep working without further changes here.
- Refactor of the surrounding `list_tools_for_client_uncached` map at `:323-355` correctly hoists `let mut tool_def = tool.tool` *before* the sanitize call so the meta mutation operates on the same object that's later moved into the returned `LoadedTool`. The reordering also rebinds `connector_id`/`connector_name`/`connector_description` from the sanitized return values rather than the original `tool.connector_*` fields — without this, the stripped meta and the unstripped struct fields would have desynced and the trusted-connector-name-normalization at `:332-336` would still fire on untrusted inputs. Easy footgun to miss; the reorder is load-bearing.
