# openai/codex PR #20724 — app-server: align dynamic tool identifiers with Responses API

- **Head SHA:** `bacaeb70603c58bce21dce8f19774130f734caf9`
- **Files:** 3 (`README.md`, `codex_message_processor.rs`, `tests/suite/v2/dynamic_tools.rs`)
- **LOC:** +221 / −1

## Observations

- `codex_message_processor.rs:8970-9019` — adding `^[a-zA-Z0-9_-]+$` validation, length caps (128 for name, 64 for namespace), and a frozen blocklist of reserved Responses namespaces (`functions`, `multi_tool_use`, `file_search`, `web`, `browser`, `image_gen`, `computer`, `container`, `terminal`, `python`, `python_user_visible`, `api_tool`, `tool_search`, `submodel_delegator`) is exactly the right shape for a defensive validator at the edge.
- `codex_message_processor.rs:8985-9007` — the helper `validate_dynamic_tool_identifier` byte-validates with `is_ascii_alphanumeric() || matches!(byte, b'_' | b'-')` and length-counts with `value.chars().count()`. Mixing byte-iteration for charset and char-counting for length is internally consistent because the regex is ASCII-only (any non-ASCII byte fails the charset check before length is counted), so the char-count is equivalent to byte-count in the only path that reaches it. Clean.
- `codex_message_processor.rs:9028,9043,9054` — error messages now route user-supplied identifiers through `escape_identifier_for_error()` (`value.escape_default().to_string()`). Good defense against log injection / terminal escape sequences in attacker-controlled tool names.
- `codex_message_processor.rs:9059-9063` — `RESERVED_RESPONSES_NAMESPACES.contains(&namespace)` uses an O(n) linear scan over a sorted 14-element slice. Fine for n=14, but if the list grows consider a `phf::Set` or a `match` statement for compile-time hashing. Non-blocking.
- Tests at `dynamic_tools.rs:241-279` and the 5 new unit tests in `codex_message_processor.rs` (`accepts_responses_compatible_identifiers`, `rejects_name_not_supported_by_responses`, `rejects_namespace_not_supported_by_responses`, `rejects_*_longer_than_responses_limit`, `rejects_reserved_responses_namespace`) cover both happy and unhappy paths well; the e2e test asserts JSON-RPC error code -32600.
- README at `README.md:1311-1316` documents the constraints — good. Lists the reserved namespaces inline so users don't have to read the source.

## Verdict: `merge-as-is`

- Validation logic is correct, well-tested, and the API contract is now clearly documented. Optional micro-perf via `phf::Set` if the reserved list grows.
