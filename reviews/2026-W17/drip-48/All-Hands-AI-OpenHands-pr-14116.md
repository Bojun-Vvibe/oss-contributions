# All-Hands-AI/OpenHands#14116 — fix: normalize legacy MCP config in migration 108

- **URL**: https://github.com/All-Hands-AI/OpenHands/pull/14116
- **Author**: neubig
- **Head SHA**: `73dcee5f01038f56119484ddd882aea0199037c8`
- **Verdict**: `merge-after-nits`

## Summary

Adds bidirectional normalization to migration 108
(`enterprise/migrations/versions/108_add_agent_settings_to_enterprise_settings.py`).
Old enterprise settings rows can carry MCP config in either the legacy
shape (`{sse_servers, stdio_servers, shttp_servers}` lists) or the new
canonical `{mcpServers: {name: config}}` map. The migration now
converts legacy → canonical on upgrade via `_normalize_mcp_config` and
provides the inverse `_to_legacy_mcp_config` for downgrade. Server name
collisions are resolved by `_next_server_name` which appends `_1`,
`_2`, … suffixes.

## Reviewable points

- `_next_server_name` (lines ~52–60) is correct but `O(n²)` in the
  pathological case of dense suffix collisions; for the realistic
  cardinality of MCP servers per workspace this is fine.
- `_normalize_mcp_config` short-circuits on the canonical form at
  lines ~67–70: if `mcpServers` is already a Mapping, it returns
  `{'mcpServers': mcp_servers}` (or `None` if empty). The empty-map →
  `None` collapse is a behaviour change worth flagging — downstream
  code that checked `config["mcpServers"] == {}` will now see `None`.
  Confirm this matches the loader's contract.
- The `sse` branch (lines ~80–88) coerces a bare-string entry to
  `{'url': entry}`. Good defensive handling. Same pattern is repeated
  for `shttp`. The `stdio` branch (lines ~101–110) does **not** apply
  the same string-coercion (it requires a Mapping with a `command`),
  which is consistent with the legacy shape but worth a comment.
- `_legacy_api_key` (lines ~115–118) treats the string `'oauth'` as a
  sentinel meaning "not a real API key". This is a leaky abstraction
  — `oauth` is a transport hint, not a key value. Consider extracting
  this into a named constant `OAUTH_AUTH_SENTINEL = 'oauth'` with a
  comment explaining why it must round-trip through the legacy shape
  as `None`.
- The downgrade path `_to_legacy_mcp_config` silently drops servers
  that don't have either a `url` or a `command` (the
  `if isinstance(server_config, Mapping): continue` filter). In a
  downgrade scenario this could lose user data without warning;
  consider logging at WARN.
- No tests visible in this diff. Migration code with bidirectional
  semantics is exactly the place where round-trip property tests pay
  off (`normalize(to_legacy(canonical)) == canonical` modulo the
  documented coercions).

## Rationale

The migration is doing real work — it converts a permissive legacy
shape into a strict canonical one and back, including handling
collisions, three transport types, and an OAuth auth hint. The logic
looks right but the absence of tests on a downgrade-capable migration
is a gap. The `oauth` sentinel and the silent-drop on downgrade are
both worth addressing before merge.
