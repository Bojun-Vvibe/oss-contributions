# openai/codex #19878 — Ingest node_repl stderr telemetry spans

- **PR**: [openai/codex#19878](https://github.com/openai/codex/pull/19878)
- **Head SHA**: `963fcb88`
- **Stats**: +1147/-5 across 12 files

## Summary

Adds an end-to-end stderr-span-telemetry ingest path so a local stdio
MCP server (specifically `node_repl`) can emit `@codex-telemetry`-prefixed
JSON spans on stderr, and have those spans reconstructed inside
codex's OTel pipeline as children of the `node_repl.js` tool-call span.
Three layers: (1) request-side `_meta` augmentation that attaches a
W3C `traceparent`/`tracestate` to outbound `node_repl/js` MCP tool
calls (`mcp_tool_call.rs:723-776`), (2) stdio-server stderr line parser
that extracts `@codex-telemetry <json>` records and routes them to a
sink callback (`rmcp_client/.../stdio_server_launcher.rs`), (3) OTel
emit helper (`codex-otel/src/stderr_span_telemetry.rs`) that validates
`v`/`type` fields, parses span/trace/parent IDs, and emits a sanitized
span with explicit parent context.

## Specific findings

- `mcp_connection_manager.rs:113,1639-1655` — telemetry sink
  registration is gated by `server_name != "node_repl"` (early return
  `None`); only the `node_repl` MCP server gets stderr-span ingest.
  Correct narrow scoping — the `@codex-telemetry` prefix is a
  contract that exists only on the node_repl side; allowing arbitrary
  MCP servers to emit OTel spans into codex's pipeline would be a
  trust escalation (any local MCP could forge spans claiming to be
  parented under any trace).
- `mcp_tool_call.rs:726-775` —
  `augment_mcp_tool_request_meta_with_otel_stderr_spans` only
  augments meta when *both* `server == "node_repl" && tool_name ==
  "js"` (`:730`); guards against future MCP servers being named
  `node_repl` without exposing the `js` tool. Pulls
  `codex_otel::current_span_w3c_trace_context()` for the active span,
  bails if no traceparent is present (`:734-738`) — so requests
  outside an active OTel span produce no telemetry meta at all rather
  than emitting placeholder zeros.
- `stderr_span_telemetry.rs:544-548,630-670` — strict validation: `v`
  must be `1` (else `UnsupportedVersion`), `type` must be `"span"`
  (else `UnsupportedType`), span name must be in an allowlist (else
  `UnsupportedSpanName`), and each ID field is parsed strictly (else
  `InvalidField(...)`). The `UnsupportedVersion` / `UnsupportedType`
  variants are deliberately separated from generic `InvalidField`
  errors so the caller can demote them to `tracing::debug!` while
  treating malformed records as `warn!` (see
  `mcp_connection_manager.rs:53-65`) — correct: a future v2 schema
  rolling out from the node_repl side shouldn't spam the codex
  warning channel.
- `stdio_server_launcher.rs` — `CODEX_TELEMETRY_STDERR_PREFIX` is
  exact `"@codex-telemetry "` (with trailing space), so a partial
  match like `@codex-telemetry-foo` won't be misclassified as a
  telemetry line and will fall through to normal stderr passthrough.
  Test at `:1358-1380` covers both the happy path and `not-json`
  rejection.
- Test coverage is thorough — `mcp_tool_call_tests.rs:651-744` pins
  that meta augmentation only fires for `node_repl`/`js` *and* only
  when an OTel span is active (`set_default(subscriber)` setup
  followed by `info_span!("mcp.tools.call").enter()`). Six additional
  `#[test]` functions in `stderr_span_telemetry.rs` exercise version
  rejection, type rejection, missing-field rejection, and successful
  span reconstruction.

## Nits / what's missing

- The allowlisted set of acceptable `name` values for incoming spans
  isn't documented in a top-level comment in
  `stderr_span_telemetry.rs`. A reader trying to add a new acceptable
  span name would have to grep the file to find the match arm. One
  comment block at the top of the file naming the contract ("the
  set of `name` values codex accepts is intentionally tiny — adding
  to this list lets the node_repl side parent any span under codex's
  trace, which is a trust grant") would help.
- The `Arc<dyn Fn(...)>` sink at
  `mcp_connection_manager.rs:52` is constructed once per
  `make_rmcp_client` call. For repeated `node_repl` connect/disconnect
  cycles in a long-running session this allocates new Arcs each time;
  not a perf problem at current scale but worth a comment that the
  closure captures only `'static` data (it does — only references the
  `codex_otel::*` module).
- Sink callback at `:53-66` ignores the return value of
  `emit_node_repl_stderr_span_telemetry` only after pattern-matching
  on the error type. A future error variant added to
  `StderrSpanTelemetryError` would need an explicit arm or fall into
  the `_ =>` warn branch — fine for now but mark with a
  `#[non_exhaustive]` or explicit `// EXPLICITLY EXHAUSTIVE` comment.
- No integration test that actually plumbs a real `node_repl` stderr
  line through the full sink → emit path. The unit tests cover each
  layer in isolation but a single end-to-end test that spawns a
  fake stdio server emitting `@codex-telemetry {...}` and asserts
  the OTel exporter sees the resulting child span would lock the
  contract end-to-end.

## Verdict

**merge-after-nits** — the design is right (narrow scope to one
named server + one named tool, strict version/type validation with
demotable error variants, W3C trace context propagation via standard
`_meta` channel) and the test coverage at each layer is solid. Nits
are mostly documentation/exhaustiveness hygiene; the missing
end-to-end test would close the contract but isn't strictly
blocking given the per-layer coverage already in place.
