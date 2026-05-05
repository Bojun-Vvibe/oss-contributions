# openai/codex #21221 — [codex] Use shared app-server internal error helper

- **Head:** `d2b49ce` (`d2b49cefdc496561ce64907dc6cbc0c38193a0cb`)
- **Repo:** openai/codex
- **Scope:** `codex-rs/app-server/src/in_process.rs`, `outgoing_message.rs`, `request_processors.rs`, `request_processors/account_processor.rs` (+ likely several other request_processors files)

## What the diff does

Mechanical refactor: every direct construction of

```rust
JSONRPCErrorError {
    code: INTERNAL_ERROR_CODE,
    message: "...".to_string(),
    data: None,
}
```

is replaced with the existing `error_code::internal_error(...)` helper. Imports of `INTERNAL_ERROR_CODE` are removed wherever the helper now covers the only use. Touches `in_process.rs:553-560, 627-660, 685-700`, `outgoing_message.rs:156-167, 1011-1015, 1137-1141, 1247-1251`, `request_processors.rs:7` (import-only), and `account_processor.rs` in 4 places (`:287-290`, `:354-360` which gets a nicer `if`/`else` matching `invalid_request`/`internal_error`, `:686-690`, `:886-890`, `:902-905`, `:935-945`).

One subtle non-mechanical spot worth pointing out: `outgoing_message.rs:156-167` — the previous direct-struct construction inlined `data: Some(serde_json::json!(...))`; the refactor reproduces this via `let mut error = internal_error(...); error.data = Some(...);` block. Semantics identical, but the `let mut` + post-mutation pattern is the only place in the diff that diverges from the simple `internal_error("msg")` substitution.

## Why it is good

Reduces 5+ places where adding fields to `JSONRPCErrorError` (e.g. trace ID, request ID echo, structured detail) would need synchronized edits. The helper is the right boundary; this PR finishes the migration that earlier PRs started. `account_processor.rs:354-360` in particular is a readability win — the old code used a single struct literal with two ternaries inside `code:` and `message:`; the new `if is_not_found { invalid_request(...) } else { internal_error(...) }` reads top-to-bottom.

## Concerns

1. **Test coverage is implicit** — the test files at `outgoing_message.rs:1011, 1137, 1247` are themselves migrated to use `internal_error("boom")` etc. That confirms the helper produces the same shape the tests already pin, but doesn't add new coverage for the helper itself. Fine for a refactor.
2. **`request_processors.rs:7` removes the `INTERNAL_ERROR_CODE` import.** Verify (probably already verified by `cargo check` in CI) that no remaining function in `request_processors.rs` still needs the bare constant — if any sub-module re-exports through this file, removal could cascade.
3. **`outgoing_message.rs:156-167` `let mut` mutation pattern** — minor: would be cleaner as `internal_error(msg).with_data(json!(...))` if the helper grew a builder method, but adding that is out of scope for this PR.

## Verdict

**merge-as-is** — pure refactor, semantics-preserving, no functional change visible to clients, tests migrated alongside. Head `d2b49ce`.
