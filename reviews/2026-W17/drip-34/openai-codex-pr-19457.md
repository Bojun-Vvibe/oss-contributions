# openai/codex #19457 — Centralize thread git metadata overlays

- **Repo**: openai/codex
- **PR**: [#19457](https://github.com/openai/codex/pull/19457)
- **Head SHA**: `ffcedfd5703d726f14dc54729eb5f62b30e5240e`
- **Author**: joeytrasatti-openai
- **State**: OPEN
- **Verdict**: `merge-after-nits`

## Context

The thread-list path in `RolloutRecorder` previously routed every state-DB
consultation through a single `get_state_db(...)` handle in
`codex-rs/rollout/src/state_db.rs`. That handle returns `None` until the
backfill check has completed, which means: on a fresh install (or after
SQLite was reset), filesystem-discovered threads come back with empty
`git`/`branch`/`worktree` overlay fields even though those fields are
already populated for the same `thread_id` in the SQLite mirror — the DB
just hadn't been declared "ready" yet. This PR splits the handle into a
two-tier abstraction (`completed` for DB-backed paging, `exact_lookup` for
per-thread metadata overlay) and routes the filesystem fallbacks through
the new `ExactThreadMetadataLookup`.

## Design

The new `ThreadListStateDb` struct in
`codex-rs/rollout/src/state_db.rs:23-51` cleanly encodes the asymmetry: a
DB that hasn't completed backfill is **not safe to page from** (you'd miss
threads), but **is safe to query by exact `thread_id`** for an overlay
because every row has a stable PK. `ExactThreadMetadataLookup` deliberately
exposes only `get_thread(thread_id)` — no `list_*` surface — which is the
right shape to enforce this at compile time.

In `codex-rs/rollout/src/recorder.rs:367-440`, the rename from
`fill_missing_thread_item_metadata_from_state_db` to
`overlay_thread_item_metadata_from_state_db` is more than cosmetic: the old
name implied "only fill when missing" but the call sites at lines 533-548
now run **even when `state_db_ctx.is_none()`** (the legacy
"SQLite-unavailable" branch), so the new name correctly advertises that
this is an unconditional overlay step. The test additions in
`codex-rs/rollout/src/recorder_tests.rs:778-848` exercise exactly this
path: filesystem hit + DB present + backfill incomplete → overlay must
still apply. Good coverage.

## Risks / Nits

1. `ExactThreadMetadataLookup` clones a full `StateDbHandle` (Arc) —
   harmless, but worth a one-liner doc comment that `runtime` here is the
   *same* underlying handle as `completed` would be when ready, just
   exposed through a narrower API. Future readers will wonder if it's a
   second connection.
2. `recorder.rs:982` rename of `page_from_filesystem_scan`'s consumer is
   fine, but the test
   `fill_missing_thread_item_metadata_preserves_filesystem_identity` at
   `recorder_tests.rs:848` was renamed only in callers — the test fn name
   itself still says "fill_missing", which now lies. Quick rename for
   grep-ability.
3. The `app-server/tests/suite/v2/thread_resume.rs:884-954` integration
   test is the right place to lock this down, but it asserts overlay
   contents only after a `thread_resume` round-trip. A negative test
   asserting "overlay does NOT happen when `exact_lookup` is `None`"
   (i.e., feature flag off) would lock the new branch invariant.

## What I learned

The interesting move here is recognizing that **"DB ready for paging"**
and **"DB ready for point lookups"** are different readiness predicates,
and that conflating them in a single `Option<Handle>` causes silent
metadata loss for users on the slow path. The fix isn't more locking or a
new RPC — it's narrowing the type that filesystem-fallback paths receive
so they can't accidentally call the unsafe operations.
