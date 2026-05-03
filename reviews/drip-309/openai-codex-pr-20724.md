# openai/codex #20724 — app-server: align dynamic tool identifiers with Responses API

- PR: https://github.com/openai/codex/pull/20724
- Author: eternal-openai
- Head SHA: `bacaeb70603c58bce21dce8f19774130f734caf9`
- Updated: 2026-05-02T02:39:46Z

## Summary
Tightens server-side validation for `dynamicTools` registered via `thread/start` so that names and namespaces match what the Responses API itself accepts: `^[a-zA-Z0-9_-]+$`, names ≤128 chars, namespaces ≤64 chars, and namespaces cannot collide with a fixed list of reserved Responses runtime namespaces (`functions`, `multi_tool_use`, `file_search`, `web`, `browser`, `image_gen`, `computer`, `container`, `terminal`, `python`, `python_user_visible`, `api_tool`, `tool_search`, `submodel_delegator`). Also expands existing whitespace-error messages to escape control characters, and ships unit + integration tests.

## Observations
- `codex-rs/app-server/src/codex_message_processor.rs:8971-9020`: the `RESERVED_RESPONSES_NAMESPACES` list is hardcoded inside `validate_dynamic_tools`. This list is effectively a contract with an external API surface that evolves. Hoist it to a module-level `const` in a colocated `responses_reserved.rs` or near the Responses request-builder code, so future Responses additions are added in one obvious place rather than buried in a validation helper. Also worth adding a `// TODO: keep in sync with <link to Responses API doc>` comment.
- Same file lines 9024-9026: `validate_dynamic_tool_identifier` uses `value.bytes().all(|byte| byte.is_ascii_alphanumeric() || matches!(byte, b'_' | b'-'))`. This correctly rejects any non-ASCII input because UTF-8 multi-byte sequences contain bytes ≥0x80, which fail `is_ascii_alphanumeric`. Good. The length check at line 9031 uses `value.chars().count()` (codepoints) — slightly inconsistent with the byte check above but fine since rejected non-ASCII inputs never reach the length check.
- Lines 9047-9061 (whitespace-trim error messages now use `escape_identifier_for_error`): nice defensive improvement — a name like `"foo\nbar "` will now show as `foo\\nbar ` rather than corrupting the JSON-RPC error payload across a literal newline. Worth applying the same `escape_identifier_for_error` to the `"dynamic tool name is reserved: {name}"` and `"dynamic tool namespace is reserved for {name}: {namespace}"` arms (lines around 9035 and 9048 in the post-diff view) for consistency — those still interpolate raw values.
- `codex-rs/app-server/tests/suite/v2/dynamic_tools.rs:240-285`: integration test exercises the full JSON-RPC error path and asserts both the error code (-32600) and that the message contains both `Responses API` and the offending identifier `lookup.ticket`. Good coverage.
- Unit tests at lines 10202-10336: cover happy path (`Codex-App_2` / `lookup-ticket_2`), invalid name dot, invalid namespace dot, oversize name (129), oversize namespace (65), and reserved `functions`. Missing: an *underscore-prefix* ASCII case like name `_internal` (allowed?), an empty-string boundary at `max_len = 0` (n/a since min length is checked elsewhere), and a Unicode-letter case like `lookupñ` to lock in the byte-level rejection. Add 1-2 of these for completeness.
- The `RESERVED_RESPONSES_NAMESPACES` array uses lowercase exact matching. If the upstream Responses API treats namespace names case-insensitively, `Functions` would slip through here but be rejected upstream. Confirm casing semantics; if case-insensitive, `eq_ignore_ascii_case` per entry.
- README/docs update at `codex-rs/app-server/README.md:1310-1314` is clear and matches the validator behavior. Good.

## Verdict
`merge-after-nits`
