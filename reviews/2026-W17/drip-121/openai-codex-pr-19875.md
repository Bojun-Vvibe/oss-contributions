# openai/codex #19875 — Strip connector provenance metadata from custom MCP tools

- URL: https://github.com/openai/codex/pull/19875
- Head SHA: `8cf1249cd6e3e9671c659df4e47da476c3c6c173`
- Diff: +142/-11 in `codex-rs/codex-mcp/src/rmcp_client.rs`.

## Context / problem

Custom (non-trusted) MCP servers were free to advertise `connector_*` metadata fields (e.g. `connector_id: "connector_gmail"`, `connector_name: "Gmail"`, plus arbitrary `connector*` keys in `tool.meta`) on their tool definitions. The agent has no way to distinguish a real Gmail connector tool published by the trusted built-in `CODEX_APPS_MCP_SERVER_NAME` from a malicious custom MCP that spoofs the same provenance — a real privilege-escalation/UI-spoofing surface (the "(via Gmail)" label users trust appears on a tool implemented by an arbitrary third party).

## What the fix does

Adds `sanitize_tool_connector_metadata(server_name, tool, connector_id, connector_name, connector_description)` (`rmcp_client.rs:369-382`) called from inside the `list_tools_for_client_uncached` per-tool `.map(|tool| { ... })` closure (`:323-336`). The sanitizer is a single-arm gate:

```rust
if server_name == CODEX_APPS_MCP_SERVER_NAME {
    return (connector_id, connector_name, connector_description);
}
strip_untrusted_connector_meta(tool);
(None, None, None)
```

For non-trusted servers it (a) returns `None` for the three connector fields so the downstream `ManagedTool` carries no provenance, and (b) calls `strip_untrusted_connector_meta` (`:384-388`) which `tool.meta.retain(|key, _| !is_connector_meta_key(key))` — the predicate is a case-insensitive prefix check on `"connector"` (`:390-393`). That correctly catches `connector_id`, `connector_name`, `connector_description`, `connectorDescription`, `connectorFutureField`, `CONNECTOR_UPPERCASE`, and any future `connector*` key the spec might add, while preserving non-connector keys like `openai/fileParams` and arbitrary user `custom: "kept"` metadata.

## Specific references

- `rmcp_client.rs:323-336` — call site. The mutable `tool_def` is bound first, sanitizer runs *before* `normalize_codex_apps_callable_name` and `normalize_codex_apps_callable_namespace` so the normalized callable name/namespace can never embed a stripped connector_name.
- `rmcp_client.rs:369-382` — `sanitize_tool_connector_metadata`. Server-allow-list against the constant `CODEX_APPS_MCP_SERVER_NAME` is the right axis (server identity, not per-tool capability).
- `rmcp_client.rs:390-393` — `is_connector_meta_key`. `key.get(..CONNECTOR_META_PREFIX.len()).is_some_and(|prefix| prefix.eq_ignore_ascii_case(CONNECTOR_META_PREFIX))` is correct UTF-8-safe prefix matching (won't panic on multibyte first chars), and case-insensitive ASCII prefix is the right policy (catches `CONNECTOR_*` and `Connector*` typo variants).
- `rmcp_client.rs:425-441` — the `custom_mcp_connector_metadata_is_stripped` test. Constructs a tool with seven connector-flavored keys plus `openai/fileParams: ["file"]` and `custom: "kept"`, asserts all three returned `Option<String>`s are `None`, iterates remaining `meta.0.keys()` asserting none match `is_connector_meta_key`, and asserts `meta.0.contains_key("openai/fileParams")` survives. Both halves of the gate are pinned.

## Risks / nits

- The test for the trusted-server pass-through path (i.e. when `server_name == CODEX_APPS_MCP_SERVER_NAME`, the metadata survives intact) is presumably present further in the file but truncated in the diff capture I have. If absent, add it — without that, a future refactor that flips the predicate's polarity goes uncaught.
- The constant `CONNECTOR_META_PREFIX = "connector"` matches things like `connectivityCheck` if any tool ever ships such a key — extremely unlikely and probably acceptable, but worth a one-line code comment naming the policy as "anything starting with 'connector' (case-insensitive) is considered provenance metadata; reserved namespace".
- This is the kind of capability-narrowing change that gets cheaper the earlier it lands; no migration concern because external custom MCPs have no legitimate reason to be advertising `connector_*` keys in the first place.

## Verdict

`merge-as-is` — server-allow-list at the lowest layer that handles tool-definition ingest, with the case-insensitive ASCII-prefix `connector*` strip catching the whole namespace, and a test pinning both the strip and the preservation of unrelated `meta` keys. Exactly the right shape for a security-relevant capability-narrowing fix.
