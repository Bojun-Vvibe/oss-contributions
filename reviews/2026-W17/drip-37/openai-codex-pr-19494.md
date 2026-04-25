# openai/codex #19494 — Streamline thread read handlers (Result-returning helpers)

- **Repo**: openai/codex
- **PR**: [#19494](https://github.com/openai/codex/pull/19494)
- **Head SHA**: `8f1b0a00ab635a891787fdeac24e8a23a462a646`
- **Author**: pakrym-oai
- **State**: OPEN (+115 / -159)
- **Verdict**: `merge-as-is`

## Context

The thread read/list handlers in `codex_message_processor.rs` had
the classic JSON-RPC handler shape: every fallible step did
`send_invalid_request_error(...).await; return;` or
`send_internal_error(...).await; return;`, so the success path was
buried under early-return arms and you had to scan past 3-4
"send + return" pairs to find the actual data assembly.

## Design

This is one of a series of mechanical refactors (see also #19490,
#19493, #19495-#19498) that converts these handlers to:

```
async fn thread_read(&self, request_id, params) {
    let result = self.thread_read_response(params).await;
    self.outgoing.send_result(request_id, result).await;
}

async fn thread_read_response(&self, params)
    -> Result<ThreadReadResponse, JSONRPCErrorError> { ... }
```

Concrete moves:

1. New error mapper `thread_read_view_error` at
   `codex-rs/app-server/src/codex_message_processor.rs:488-493`
   collapses the `match err { InvalidRequest(...) => ..., Internal(...) => ... }`
   pattern into one helper used by `read_thread_view` callers.
2. `thread_loaded_list` (lines 3633-3705): empty-data short-circuit
   becomes an `Ok(...)` return; cursor-parse failure becomes
   `return Err(invalid_request(...))`. The control flow now reads
   straight through.
3. `thread_read` (lines 3707-3736): `ThreadId::from_string` →
   `.map_err(|err| invalid_request(format!("invalid thread id: {err}")))`.
   `read_thread_view` → `.map_err(thread_read_view_error)?`.
4. The same shape applied to `thread_turn_list` and
   `thread_summary` handlers.

The `send_result` outgoing helper (already present in the
codebase from earlier PRs in the series) handles the
`Result<T, JSONRPCErrorError>` → wire dispatch switch.

## Risks

- **Behavior preservation.** The InvalidRequest vs Internal
  distinction is preserved because the mapping function is
  pattern-aligned with the original early-return arms. The
  `send_invalid_request_error` and `send_internal_error` helpers
  produced exactly the same error codes that `invalid_request()`
  and `internal_error()` produce now (they share the
  `INVALID_REQUEST_ERROR_CODE` / `INTERNAL_ERROR_CODE` constants).
- **No data field changes.** Both old and new paths pass `data: None`,
  so no observable wire diff.
- **Test coverage.** The author ran
  `cargo test -p codex-app-server --test all conversation_summary -- --test-threads=1`
  which exercises summary + thread-read paths. The mechanical
  shape means existing integration tests are the right gate.

## Suggestions

None. This is exactly the right kind of refactor: net negative
LOC (-44), no behavior change, and pulls handler shape into
alignment with the rest of the series.

## What I learned

Worth noting that this series only works cleanly because the
early returns were *pure dispatch* — no side effects between the
fallible step and the `send_error`. If any of these handlers had
been doing partial state mutation before the error point (e.g.
inserting a subscription record then bailing), the
Result-returning helper rewrite would have had to extract a
guard/cleanup path. The PR series picks handlers in dependency
order presumably for exactly this reason.
