# openai/codex#20562 — Use the 2025-06-18 elicitation capability shape

- **Author:** Abhinav (abhinav-oai)
- **Head SHA:** `1e4430a4fbfa6db21e4b89f9583e83fcaa138d81`
- **Base:** `main`
- **Size:** +7 / -18 across 2 files
- **Files changed:** `codex-rs/codex-mcp/src/connection_manager_tests.rs`, `codex-rs/codex-mcp/src/rmcp_client.rs`

## Summary

Spec-compliance fix for MCP elicitation capability advertisement. The 2025-06-18 MCP spec at modelcontextprotocol.io explicitly says the `elicitation` capability "should be an empty object" (already noted in a code comment at `rmcp_client.rs:323-324`), but the implementation was advertising the richer shape `{form: {schema_validation: null}, url: null}` from a later draft. PR replaces the construction with `ElicitationCapability::default()` which serializes to `{}`, matching the spec.

## Specific code references

- `rmcp_client.rs:323-325`: the load-bearing change — `Some(ElicitationCapability { form: Some(FormElicitationCapability { schema_validation: None }), url: None })` collapsed to `Some(ElicitationCapability::default())`. The pre-existing comment at `:323-324` (`// https://modelcontextprotocol.io/specification/2025-06-18/client/elicitation#capabilities indicates this should be an empty object.`) was already documenting the correct intent — the prior code was the bug.
- `rmcp_client.rs:58` (deletion): removes the now-unused `use rmcp::model::FormElicitationCapability;` import. Symmetric cleanup with the test file.
- `connection_manager_tests.rs:803`: test renamed from `elicitation_capability_enabled_for_custom_servers` to `elicitation_capability_uses_2025_06_18_shape_for_all_servers` — the rename itself is meaningful documentation of what's being asserted.
- `connection_manager_tests.rs:805-810`: the new assertion shape pins the contract via two predicates: `assert_eq!(capability, Some(ElicitationCapability::default()))` for the Rust-side equality, and crucially `assert_eq!(serde_json::to_value(capability).expect(...), serde_json::json!({}))` for the wire shape. The second assertion is the load-bearing one — it would catch a future `rmcp` version that adds `Default`-non-`None` fields to `ElicitationCapability` and silently breaks spec compliance even if the Rust code stays the same.
- `connection_manager_tests.rs:29` (deletion): drops the `use rmcp::model::FormElicitationCapability;` import; test no longer references the inner-form type at all, which is the right cleanup.
- `connection_manager_tests.rs:802` test loops over both `CODEX_APPS_MCP_SERVER_NAME` and `"custom_mcp"` — preserves the "all servers, not just custom" coverage from the prior test name, important because `elicitation_capability_for_server` could regress to a per-server-name branch.

## Reasoning

This is the cleanest possible spec-compliance fix: the implementation already had a comment pointing at the spec section it was violating, and the fix is one constructor call swap. The test reformulation is the genuinely good part — pinning by `serde_json::to_value(...) == json!({})` rather than just structural equality means future `rmcp` upstream changes that add fields to `ElicitationCapability` will fail loudly here rather than silently regress wire compatibility.

The only thing that could have been added is a comment at the new construction site cross-referencing the test name, so a future contributor who tries to "enrich" the capability shape gets steered to the spec link. But the comment at `:323-324` already does that adequately.

## Verdict

**merge-as-is**
