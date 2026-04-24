# openai/codex #19377 — feat: separate session_id and thread_id in Responses requests

**Link:** https://github.com/openai/codex/pull/19377
**Tag:** schema-drift, multi-agent-v2

## What it does

Splits one identifier into two on the Responses API wire. Multi-agent
v2 needs (a) a session-level id grouping the whole thread tree and
(b) a thread-level id telling the backend which child thread is
handling the current turn. Before this change, Codex reused the current
thread id as `session_id`, so spawned children couldn't share the root
session identity while still identifying themselves distinctly.

Concretely:

- `ResponsesOptions.conversation_id: Option<String>` becomes two fields,
  `session_id` and `thread_id`.
- `build_conversation_headers(Option<String>)` becomes
  `build_session_headers(Option<String>, Option<String>)` — emits
  `session_id` and `thread_id` headers separately.
- `x-client-request-id` is now keyed off `thread_id` (the current child),
  not the session.
- `ModelClientState` carries both `root_thread_id: ThreadId` and
  `thread_id: ThreadId`. `ModelClient::new` takes both. The cache window
  id (`current_window_id`) keys on `thread_id`, not the session — so
  child threads get their own SSE/websocket cache slot.
- Compaction and Responses paths both extend extra headers with the new
  pair.

## What it gets right

- **Header rename, not header overload.** Adding `thread_id` instead of
  multiplexing two values into `session_id` keeps both old and new
  semantics legible on the wire — backend logs are unambiguous about
  which child handled which turn.
- `current_window_id` keyed on `thread_id` is the correct change: the
  SSE retry/window cache must not collide across sibling child threads
  that share a root session, which is exactly what the old
  conversation-keyed window would have done under v2.
- Test coverage extends the existing
  `azure_default_store_attaches_ids_and_headers` to assert both headers
  land — small but it pins the wire contract.
- Clean rename through `codex-api` re-exports
  (`build_conversation_headers` → `build_session_headers`) catches all
  consumers at compile time.

## Concerns / risks

- **Breaking signature on `ModelClient::new`.** The function gained a
  required `root_thread_id: ThreadId` parameter. Internal-only crate
  but worth noting for any downstream embedders. No deprecation
  shim/builder offered.
- **No backend back-compat handling.** If a backend that only knows
  `session_id` receives `session_id=root` + `thread_id=child` and
  internally routes by session, child-thread state may bleed across
  siblings. The PR description says "live spawn flows only" and
  reopening is out of scope, but doesn't note any backend-version gate
  — silent backend behavior change risk.
- The PR keeps `parent-thread headers` for spawned children "preserved"
  but doesn't show that path in the diff; reviewer needs to trust the
  wider commit history. Worth a comment in `client.rs` explicitly
  pointing at `subagent_header` + parent-thread plumbing so the next
  reader doesn't reintroduce the conflation.
- Reopening / lineage reconstruction is explicitly excluded, leaving
  thread resume on a different code path with the **old** single-id
  semantics. That asymmetry is a future foot-gun: a child thread
  resumed offline may rejoin with `session_id == thread_id`, which
  the backend will then have to re-disambiguate.

## Suggestion

Add a `debug_assert_ne!(root_thread_id, thread_id)` (or at least a
`tracing::debug!`) at the spawn site so the spawn path can't silently
fall back to the old "same id for both" shape under refactoring drift.
The whole point of the PR is that those two ids are distinct on
spawn — make that invariant load-bearing in code, not just intent.
