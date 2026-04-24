# openai/codex #19207 — Forward Codex Apps tool call IDs to backend metadata

**Link:** https://github.com/openai/codex/pull/19207
**Tag:** observability, contract-pairing

## What it does

Threads the outer tool `call_id` through `build_mcp_tool_call_request_meta`
and writes it into `_meta._codex_apps.call_id` for tool calls targeting
the Codex Apps MCP server. Pairs with a backend change that prefers
`_meta._codex_apps.call_id` over the JSON-RPC request id when logging
MCP compliance — so the backend can correlate model/tool call ids with
MCP traffic without depending on transport-layer rpc ids.

Implementation is small:

- `build_mcp_tool_call_request_meta` gains a `call_id: &str` parameter.
- For `CODEX_APPS_MCP_SERVER_NAME`, it now constructs `codex_apps_meta`
  unconditionally (`metadata.codex_apps_meta.clone().unwrap_or_default()`)
  and inserts `call_id` into it. Previously the whole branch was gated
  on `metadata.codex_apps_meta` being `Some`, so calls without
  pre-existing apps metadata wrote no `_codex_apps` block at all.
- Test coverage adds a new
  `codex_apps_tool_call_request_meta_includes_call_id_without_existing_codex_apps_meta`
  case alongside the existing "with metadata" case. Two integration
  tests in `openai_file_mcp.rs` and `search_tool.rs` are updated to
  assert `call_id` lands in the on-the-wire payload.

## What it gets right

- **Wire-compatible.** `_meta._codex_apps` is reserved
  backend-only metadata; older backends ignore the new `call_id`
  field. No coordinated rollout needed.
- The unwrap-default refactor closes a real gap: previously, a Codex
  Apps tool call **without** prior approval metadata wrote *no*
  `_codex_apps` block, which means the new `call_id` correlation point
  would have been missing exactly when there was nothing else to
  correlate on. Both branches now consistently emit the block.
- Test surface covers both the merge case (existing apps meta +
  call_id) and the empty-default case. The two end-to-end tests
  (`openai_file_mcp`, `search_tool`) prove the field actually reaches
  the serialized JSON-RPC params.

## Concerns / risks

- **Always emitting `_codex_apps` for the apps server, even with no
  other meta**, broadens the backend's surface from "presence of block
  signals approved/connector context" to "block always present, fields
  inside signal context". Backends doing
  `if "_codex_apps" in meta:` (without checking nested fields) will
  now hit the branch on every call. Should be benign given the paired
  backend change, but worth flagging.
- `call_id` is stringly-typed and forwarded as-is. If the
  outer `call_id` ever contains operator-controlled or model-controlled
  content with funny characters (newlines, JSON-escape edge cases),
  it'll land in MCP compliance logs verbatim. A length cap or
  character-set assert at construction would be defensive.
- The function signature gained a **fourth** positional `&str`
  parameter sandwiched between `&str` (server) and an `Option<&...>`
  (metadata). Easy to mix up at call sites under future refactor; a
  `BuildMcpRequestMeta { ... }` struct would make the call sites
  self-documenting.

## Suggestion

Promote the four positional args to a small builder/struct, or at
minimum rename the param `outer_call_id: &str` and add a doc comment
distinguishing it from the JSON-RPC `request_id` it's intended to
replace in compliance logs. The whole point of this change is that
those two ids are different — the code should make that explicit.
