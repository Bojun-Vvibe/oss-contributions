# openai/codex PR #20334 — Make missing config clears no-ops

- PR: https://github.com/openai/codex/pull/20334
- Head SHA: `f8059e4ce55f861c8b7e5a34bb143aec9b45604c`
- Files touched: 2 (`codex-rs/app-server/src/config_manager_service.rs` +6/-12, `codex-rs/app-server/src/config_manager_service_tests.rs` +24/-0). Net +30/-8.

## Specific citations

- Three call-sites in `codex-rs/app-server/src/config_manager_service.rs` change from "return `MergeError::PathNotFound` → propagate as `ConfigPathNotFound` write error" to "treat as a successful no-op". Specifically `clear_path` at `:480-491`:
  - Iterating `parents`: `table.get_mut(segment).ok_or(MergeError::PathNotFound)?` → `let Some(next) = table.get_mut(segment) else { return Ok(false); }; current = next;` at `:483-487`.
  - Non-table-on-the-way: `_ => return Err(MergeError::PathNotFound)` → `_ => return Ok(false)` at `:489`.
  - Final segment's parent not a table: `return Err(MergeError::PathNotFound)` → `return Ok(false)` at `:491-492`.
- The `MergeError::PathNotFound` arm in `apply_merge`'s error mapper at `:244-249` is removed. The enum at `:413-415` drops the `PathNotFound` variant entirely, leaving only `Validation(String)`. This is a clean type-level guarantee that no caller can ever produce a `ConfigPathNotFound` error from `clear_path` again.
- New regression at `codex-rs/app-server/src/config_manager_service_tests.rs:106-130` (`clear_missing_nested_config_is_noop`) writes an empty config file, sends `write_value` with `key_path: "features.personality"` and `value: serde_json::Value::Null` (which encodes a clear) plus `merge_strategy: MergeStrategy::Replace`, and asserts three things: response status `WriteStatus::Ok`, `overridden_metadata == None`, and the file contents remain `""` (no spurious table creation).

## Verdict: merge-as-is

## Concerns / nits

1. **Idempotent clear is the right semantics.** Removing a key that wasn't there should not be an error — this matches POSIX `rm -f`, `DELETE` HTTP semantics on non-existent resources (RFC 9110 allows 200/204), and what every config-management tool does. The previous behavior was a foot-gun for any UI that built a "reset to default" button: a user clicking it on a setting they never customized would get an error.
2. **Three returns of `Ok(false)`** convey the "nothing was removed" signal up the call chain. The function signature is `Result<bool, MergeError>` and the existing return at `:495` (`Ok(parent.remove(last).is_some())`) already returns `false` when the leaf wasn't present. So callers that want to distinguish "removed something" vs "nothing to remove" still can — they just won't see an error for the latter. Good API shape preserved.
3. **The regression test could be slightly stronger** by asserting that a *partial* path (e.g., `features.foo.bar.baz` where `features.foo` is a string, not a table) also returns `Ok` instead of an error. The `_ => return Ok(false)` at `:489` covers that branch but isn't exercised by the new test.
4. **No CHANGELOG/release-note entry visible in the diff slice.** This is a user-facing behavior change for any external integrations driving `write_value` programmatically — they may be relying on receiving `ConfigPathNotFound` as a signal. Worth a one-line note in the next release's notes.
