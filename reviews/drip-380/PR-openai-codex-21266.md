# openai/codex PR #21266 — [codex] Fix pathless thread summaries

- URL: https://github.com/openai/codex/pull/21266
- Head SHA: `a8a0878cf0ec975f8223d057bb1f5d67dc3e733c`
- Size: +159 / -30

## Summary

Loosens the `ConversationSummary` contract from `path: PathBuf` to `path: Option<PathBuf>` so that threads stored in non-filesystem stores (the new `InMemoryThreadStore` and presumably any future remote thread store) can be summarized at all. Previously `summary_from_stored_thread` returned `Option<ConversationSummary>` via `let path = thread.rollout_path?;` (`thread_processor.rs:3574`), so a thread with no rollout path forced the caller to either return `internal_error("…thread is missing rollout path")` (deleted at `thread_processor.rs:3072-3076`) or fail unarchive (deleted at `thread_processor.rs:1535-1537`). Now the path is propagated as `None` and the rest of the summary fields populate normally.

## Specific findings

- `codex-rs/app-server-protocol/src/protocol/v1.rs:92` — `pub path: Option<PathBuf>` — the load-bearing contract change. `ConversationSummary` is exposed via the v1 wire protocol (and re-exported to TypeScript at `schema/typescript/ConversationSummary.ts:8`), so this is a downstream-visible breaking change for any client that consumed `path` as a non-nullable string. Acceptable here because (a) the TS schema is co-versioned and (b) the docs note at `docs/codex_mcp_interface.md:55` explicitly tells `rolloutPath` lookup users that "lookups by rolloutPath won't work with non-local thread stores", flagging the new shape.
- `codex-rs/app-server/src/request_processors/thread_processor.rs:3563-3578` — `summary_from_stored_thread` is now infallible. The deleted `Option` wrapping eliminates a "summary doesn't exist" failure mode that was always semantically a "path doesn't exist" failure mode. Cleaner.
- `codex-rs/app-server/src/request_processors/thread_processor.rs:1535-1538` — `unarchive_thread` `.and_then(...).ok_or_else(internal_error("failed to read unarchived thread {thread_id}"))` collapses to `.map(|stored_thread| { let summary = ...; summary_to_thread(...) })`. This removes the previous "unarchive succeeded but we couldn't summarize it" error path entirely — that error was only ever triggered when the unarchived thread had no rollout path, which is now a normal state. Correct.
- `codex-rs/app-server/src/request_processors/thread_processor.rs:3066-3072` — `get_conversation_summary` similarly drops the `internal_error("failed to load conversation summary: thread is missing rollout path")` branch. Net behavior change: callers that previously got an error for pathless threads now get a successful response with `summary.path == null` — clients must handle the `null` case. The new TS test fixture at `tests/suite/conversation_summary.rs:226-229` (`assert_eq!(summary.path, None);`) pins exactly this contract.
- `codex-rs/app-server/src/request_processors/thread_summary.rs:257-263` — `warn!` now also includes `conversation_id` and renders `path` as `path.as_ref().map(|path| path.display().to_string()).as_deref()`, preserving observability when the path is `None`. Good (the previous `path = %path.display()` would have crashed-or-panicked on a missing path; now it simply logs `path = None`).
- `codex-rs/app-server/tests/suite/conversation_summary.rs:146-228` — new 80-line integration test `get_conversation_summary_by_thread_id_reads_pathless_store_thread` constructs an `InMemoryThreadStore`, creates a thread with no rollout path, spins up the app-server `in_process` and asserts the round-trip returns `summary.path == None`. Good end-to-end pin. The bespoke `InMemoryThreadStoreId` RAII guard (`:262-269`) calling `InMemoryThreadStore::remove_id` on `Drop` to prevent test cross-contamination is the right pattern but worth a `// keep store registry clean across parallel tests` comment.

## Concerns / nits

- **Wire-protocol break, downstream consumers**: any external client that deserializes `ConversationSummary.path` as a non-nullable `string` will fail. The PR doesn't enumerate downstream consumers (TUI, web UI, IDE extensions). Recommend a CHANGELOG note + maintainer ack on which clients have been updated to accept `null`.
- `summary_from_state_db_metadata` at `thread_processor.rs:3640` and `thread_summary.rs:68,117` now uniformly wrap the existing `path: PathBuf` as `path: Some(path)`. Consistent — no semantic change for the on-disk path.
- `summary_to_thread`'s prior `path: Some(path)` at `thread_summary.rs:276` collapses to `path` directly because `path` is already `Option<PathBuf>` upstream. Correct.
- `RAISE InMemoryThreadStoreId { store_id }` at `tests/suite/conversation_summary.rs:184` — bound to `_in_memory_store` to keep the guard alive for the test scope; this is fine but easy to misread. A comment helps.

## Verdict

`needs-discussion` — the v1 wire-protocol shape change for `ConversationSummary.path` (now nullable) is the kind of contract drift that needs explicit maintainer sign-off on which downstream consumers were audited and whether a CHANGELOG/migration note is required, even though the implementation itself is clean.
