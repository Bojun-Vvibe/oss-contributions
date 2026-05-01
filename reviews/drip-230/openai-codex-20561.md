# openai/codex #20561 — state: pass state db handles through consumers

- **PR**: https://github.com/openai/codex/pull/20561 (DRAFT)
- **Head SHA**: `111ea8a05331ec426fc8ac096b4c68d558f503ab`
- **Files reviewed**: `codex-rs/app-server/src/codex_message_processor.rs`, `codex-rs/app-server-client/src/lib.rs`, plus thread-store / rollout / TUI / exec / MCP server wiring
- **Date**: 2026-05-01 (drip-230)

## Context

SQLite state was being opened from several consumer paths — including
`OnceCell`-backed lazy thread-store call sites and ad-hoc `StateRuntime::init(...)` calls
inside `memory_reset_response` etc. — which meant a single Codex process could end up
holding multiple connections to the same `state.db` for the same `codex_home`. SQLite
serializes writers per-database-file, so multi-handle in one process is the textbook
recipe for `database is locked` errors and lock-thrash latency.

The PR moves the state DB lifetime up to main-like entrypoints (CLI / TUI / app-server /
exec / MCP server / thread-manager-sample), constructs **one** `StateDbHandle` per
process, and threads `Option<StateDbHandle>` through every consumer that previously
opened its own. Consumers that already have a fallback (filesystem-only rollout listing,
in-memory thread store) keep that fallback — they just no longer call
`get_state_db(&self.config).await` themselves to lazily open a second handle.

## Diff highlights

- `app-server/src/codex_message_processor.rs:528, 690, 819-829`: `CodexMessageProcessor`
  gains a `state_db: Option<StateDbHandle>` field, threaded through
  `CodexMessageProcessorArgs` → `Self { state_db, ... }`.
- The pattern at `:2988`:
  ```diff
  - if let Some(state_db_ctx) = get_state_db(&self.config).await {
  + if let Some(state_db_ctx) = self.state_db.as_ref() {
  ```
  is repeated at `:3309-3311` (memory_reset cancellation), `:4474` (resumed-history
  metadata reload), and `:4612` (direct thread lookup fallback). Same shape, same
  intent — replace lazy-open with caller-supplied handle.
- `:3250-3257` `memory_reset_response` previously called
  `StateRuntime::init(self.config.sqlite_home.clone(), ...)` *inside* the request
  handler; the diff replaces that with
  ```rust
  let state_db = self.state_db.clone()
      .ok_or_else(|| internal_error("sqlite state db unavailable for memory reset"))?;
  ```
  Right call: memory reset has no filesystem fallback (it must touch the DB), so
  hard-erroring when the handle isn't supplied is correct, vs. the old behavior of
  silently spinning up a second handle.
- `:3447-3473` rewrites the `find_thread_path_by_id_str` /
  `find_archived_thread_path_by_id_str` chain to the new
  `_with_state_db(..., self.state_db.as_deref())` variants — so rollout-path lookup uses
  the existing state DB index when present and falls back to filesystem scan when not.
- `app-server-client/src/lib.rs:336-338, 399, 982, 2053, 2093`: `InProcessClientStartArgs`
  gains `pub state_db: Option<StateDbHandle>`, threaded into `InProcessStartArgs`. All
  three test sites (`:982`, `:2053`, `:2093`) explicitly pass `state_db: None`, which is
  the correct "tests opt out of shared state DB" default — those tests run in isolated
  tmpdir codex_homes and don't need cross-test connection sharing.

## Observations

1. **The "remove the lazy local state DB wrapper from the thread store" line in the PR
   description is the load-bearing simplification.** Without it, `LocalThreadStore` would
   keep its `OnceCell<StateRuntime>` and the second-handle hazard would persist for any
   call path that didn't get refactored. Removing the wrapper means the type system now
   forces every consumer to either (a) accept a `StateDbHandle` from above, or (b) work
   without one — there's no longer a hidden third option.

2. **`Option<StateDbHandle>` (not `StateDbHandle`) is the right shape.** Several
   consumers (rollout listing, scan-only paths) genuinely don't need DB access and should
   keep working when SQLite is unavailable for whatever reason (corrupted file, perm
   error, in-memory test rig). The `Option` shape lets the call sites be explicit about
   "DB-required" (`.ok_or_else(...)`) vs "DB-helps" (`.as_ref()` then a fallback path).

3. **`get_conversation_summary` rewrite at `:5125-5180`** is the most intricate hunk —
   it now downcasts `self.thread_store.as_any()` to `LocalThreadStore` and, when that
   succeeds, takes a state-DB-aware path that locates the rollout file via
   `find_thread_path_by_id_str_with_state_db`. The downcast is fine as a recognition
   gate (only `LocalThreadStore` has `read_thread_by_rollout_path`), but the *fallback*
   branch when the downcast fails still needs to call the original
   `self.thread_store.read_thread(...)` path — confirm in the full diff that the `else`
   arm preserves the pre-PR behavior; if it doesn't, remote thread stores would silently
   start returning "not found" for valid IDs.

4. **`state_db: Option<StateDbHandle>` + `Clone` cost.** `StateDbHandle` is presumably
   an `Arc`-backed handle (it's cloned freely at `:3253`, `:3309`, `:4612`). If it isn't,
   the per-request `.clone()` would copy the underlying connection pool descriptor and
   the PR would actively regress what it's trying to fix. Worth confirming the
   `StateDbHandle` definition is `Clone via Arc`.

## Risks / nits

- **DRAFT, and the PR body explicitly says** `cargo test -p codex-core` was attempted but
  the linker SIGBUS'd because the workspace target filesystem filled. So the
  full-workspace test sweep hasn't actually run end-to-end — `cargo check --tests` only
  type-checks, it doesn't execute. Real risk that one of the threaded-through call sites
  has a runtime bug (e.g. `unwrap()` on an `Option` that the new shape can now make
  `None`) that won't show up until the full test pass. Don't merge until that runs
  green on a clean target dir.
- **Thread-manager-sample is included in the validation list** (`cargo check -p
  codex-thread-manager-sample`) but not in the `just fix` list — confirm formatting/clippy
  ran for it too, since it's now constructing a `StateDbHandle` itself.
- **Cross-process state DB concurrency is unchanged.** The PR fixes intra-process
  multi-handle thrash, but two `codex` processes sharing the same `codex_home` still
  open separate SQLite connections and still rely on SQLite's file-level locking. That's
  out of scope for this PR but worth a docstring on `StateDbHandle::new` ("one handle per
  process; concurrent processes serialize at the SQLite WAL layer").
- **`internal_error("sqlite state db unavailable for memory reset")`** at `:3253` is a
  silent-class regression risk: if any deployment shape was relying on the old behavior of
  "handler lazily opens a state DB if config has one", they'll now see the error instead
  of the operation working. Surface this in the changelog as a behavior change for the
  app-server entry point — must construct with `state_db: Some(...)` to support
  memory-reset.

## Verdict

needs-discussion
