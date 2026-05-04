# openai/codex PR #20949 — [codex] add thread_origin to thread analytics

- **Link:** https://github.com/openai/codex/pull/20949
- **Head SHA:** `78065f4af17767337a52bba1d2d8ee4b3a04ddce`
- **Author:** (codex team)
- **Files:** `codex-rs/analytics/src/{events.rs, reducer.rs, analytics_client_tests.rs, client_tests.rs}` plus downstream test fixtures
- **Diff size:** ~1882 diff lines (mostly test fixture updates)

## Summary
Adds a new `thread_origin: Option<String>` field to the analytics `Thread` struct, threaded through `ThreadInitializedEventParams`, `ThreadMetadataState`, and the `from_thread_metadata` constructor. The reducer now carries `thread_origin` alongside `thread_source` so that the analytics backend can distinguish the originating surface (e.g. `app_helper`) from the broader session source.

## Specific citations
- `codex-rs/analytics/src/events.rs:114` — new field `pub(crate) thread_origin: Option<String>` on `ThreadInitializedEventParams`.
- `codex-rs/analytics/src/events.rs:727` — `subagent_thread_started_event_request` initializes `thread_origin: None` (correct: subagent threads inherit, do not originate).
- `codex-rs/analytics/src/reducer.rs:150-152` — `ThreadMetadataState` gains `thread_origin: Option<String>`.
- `codex-rs/analytics/src/reducer.rs:158-179` — `from_thread_metadata` signature now takes `thread_origin: Option<String>` and stores it.
- `codex-rs/analytics/src/reducer.rs:355` — `get_or_insert_with` initializer for the subagent path also sets `thread_origin: None`. Symmetric with `events.rs:727`. Good.
- `codex-rs/analytics/src/analytics_client_tests.rs:137,848,880` — fixtures updated; one test asserts the `thread_origin: "app_helper"` round-trips through serialization.
- `codex-rs/analytics/src/client_tests.rs:90` — `sample_thread` updated to `thread_origin: None`.

## Observations
- The serialization shape (`analytics_client_tests.rs:880`) emits `thread_origin: "app_helper"` as a top-level event field. Confirm with the analytics ingest schema that this is an additive change — older consumers reading the same event must tolerate the new key.
- `Option<String>` is the right choice over `&'static str` since values come from clients (not a closed enum). However, the existing `thread_source` field uses `&'static str`. The asymmetry is justified but worth a comment in `events.rs` so future contributors do not "fix" the inconsistency.
- All `None` defaults are wired in the right places — I checked subagent paths (line 727 + line 355) and the standalone-thread path. No silent drops.
- Test coverage is good: at least one positive test (`"app_helper"`) and the implicit `None` cases throughout the existing fixtures.

## Nits
- Add a one-line doc comment on the `thread_origin` field at `events.rs:114` describing accepted values (or pointing at the schema doc) so it does not become a dumping ground.
- The reducer constructor at `reducer.rs:158` now takes three positional args; consider a small builder or named-struct param to avoid future arg-ordering bugs when more metadata fields are added.

## Verdict
`merge-after-nits`
