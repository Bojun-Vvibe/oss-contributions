---
pr_number: 19280
repo: openai/codex
head_sha: 6addbab3deb6aa0366ad83be355d79810d9ae077
verdict: merge-after-nits
date: 2026-04-25
---

# openai/codex#19280 — migrate `thread/turns/list` to `ThreadStore`

**What changed.** +1492 / −545. Three things bundled together: (1) `app-server/src/codex_message_processor.rs:404` adds a new `thread_store_override: Option<Arc<dyn ThreadStore>>` constructor arg threaded through `InProcessClientStartArgs` (`app-server-client/src/lib.rs:403`) so tests/embedders can inject a `LocalThreadStore` without touching `experimental_thread_store_endpoint`; (2) `configured_thread_store` (line ~660) factors into `configured_local_thread_store` + endpoint check; (3) the meat — `thread/turns/list` no longer resolves a rollout *path* on disk via the four-fallback chain (`resolve_rollout_path` → `find_thread_path_by_id_str` → `find_archived_thread_path_by_id_str` → live `thread_manager.get_thread().rollout_path()`) and instead goes through `ThreadStore::read_thread(...include_history: true)`. The ~95-line block at lines 4262–4357 of the prior file is gone, replaced by a store call. `summary_to_thread` is replaced by `thread_from_stored_thread`, and `thread/list` (line 3961) now iterates `StoredThread` directly with `fallback_provider: self.config.model_provider_id.clone()`.

**Why it matters.** The endpoint was leaking rollout-storage assumptions into the protocol layer: pathless stores (e.g. a remote `RemoteThreadStore` or a future SQLite-backed `LocalThreadStore`) couldn't satisfy the path-based lookup at all, and the four-stage path-fallback chain meant any one of them returning `None` while another succeeded created subtle determinism bugs across home/codex_home configurations. Active turn merging and pagination semantics are explicitly preserved, per body.

**Concerns.**
1. **`thread_store_override: Option<...>` is opaque dependency injection.** It's a single-purpose hook for tests and the in-process client — fine — but there's no compile-time guard that production paths can't pass `Some(...)`. A `ThreadStoreSource::Configured | TestOverride(Arc<...>)` enum would make the test-only intent explicit. Not blocking.
2. **`load_live_thread_for_read` deleted (formerly line 4130)** and its body inlined at line 4104 as `self.thread_manager.get_thread(thread_id).await.ok()`. Fine, but the deleted helper had a meaningful name; the inlined `.ok()` swallows the `CodexErr` without a warn-log. If the live thread fetch fails for an unexpected reason (not just "not found"), we now silently fall through to the persisted path — with no telemetry that the live path failed. Add `tracing::debug!` on the error before discarding.
3. **`thread/list` `fallback_provider` propagation.** Line 3987 captures `self.config.model_provider_id.clone()` *once* outside the loop, then passes the same value to every `thread_from_stored_thread` call. If the config is hot-reloaded mid-iteration the loop sees the stale provider; for short-lived list calls this is fine, but document it (`// snapshotted; safe because list is non-streaming`).
4. **`StoreReadThreadByRolloutPathParams` is imported (line 357)** but I don't see it used in the diff window. Either it's used in a deeper section the diff truncated, or it's a leftover import. Run `cargo check` with `-W unused-imports`.
5. **+1492 / −545 is a big swing for a "migration" PR.** Confirm the +1492 isn't dominated by new test fixtures; if any production code paths gained semantics beyond the migration (e.g. new error types, new metadata fields on `Thread`), they should be called out in the body. The body lists three bullets and nothing else.
6. **`token_usage_replay::latest_token_usage_turn_id_from_rollout_path` is removed from imports (line 408)** — confirm no other callers remain. If it's dead-on-arrival, delete the function in the same PR rather than leaving a graveyard symbol.

Direction is right (pathless-store readiness). Land after the live-fetch debug log and confirming no dead-symbol residue.
