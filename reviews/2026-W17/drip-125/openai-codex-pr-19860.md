# openai/codex #19860 — feat: split memories part 2

- **PR**: [openai/codex#19860](https://github.com/openai/codex/pull/19860)
- **Head SHA**: `a650996b`

## Summary

Second installment of the memories-package split. Continues moving the memory
*write* trigger out of `core` so the core crate becomes fully independent of
`codex-memories-write`. The dependency edge is reversed: `app-server` and
`exec` (the consumers that own a session lifecycle) now depend on
`codex-memories-write` and call into it; `core` no longer does. New
`SessionSource::Internal(InternalSessionSource::MemoryConsolidation)` variant
distinguishes memory-consolidation-driven sub-sessions from user/CLI sources
so analytics + ABOM (agent-bill-of-materials) generators can opt them out of
user-facing surfaces.

## Specific findings

- `Cargo.lock` (and the workspace dep graph it reflects) — `codex-app-server`
  and `codex-exec` gain `codex-memories-write` deps; `codex-core` *loses*
  both `codex-memories-write` and `codex-secrets`. This is the headline
  invariant of the split. The `codex-memories-write` crate itself gains a
  large set of internal deps (`codex-config`, `codex-core`, `codex-features`,
  `codex-login`, `codex-otel`, `codex-rollout`, `codex-rollout-trace`,
  `codex-secrets`, `codex-state`, `codex-terminal-detection`, `core_test_support`,
  plus `wiremock`/`tempfile`/`futures`/`pretty_assertions`) — correct for a
  crate that now owns its own end-to-end memory-write workflow including
  HTTP-mocked tests, but it does mean the crate is no longer "leaf-like".
- `app-server-protocol/schema/typescript/SessionSource.ts` — TS union widened
  with `{ "internal": InternalSessionSource }`; new
  `InternalSessionSource.ts` is a single-variant union
  `"memory_consolidation"`. Schema is generated via `ts-rs` so the change is
  mechanical, but worth noting the union now has a "future-internal-sources"
  shape that lets consumers branch without re-parsing.
- `app-server-protocol/src/protocol/v2.rs:1998-2002` — `From<CoreSessionSource>
  for SessionSource` now maps `Internal(_) => SessionSource::Unknown` with the
  comment "We do not want to render those at the app-server level." Right
  call: app-server's `SessionSource` is the wire format the IDE/UI sees, so
  bleeding internal-source kinds through there would force every UI to learn
  them. The `Unknown` mapping is pragmatic but should also bear a debug-log
  on the conversion so the engineer who later wonders "why does my memory
  consolidation session show as Unknown in the UI" finds the trail.
- `agent-identity/src/lib.rs:325` and `analytics/src/reducer.rs:109` — both
  exhaustive `match` arms gain `SessionSource::Internal(_)` next to the other
  catch-all variants (`Custom(_)`, `Unknown`). Mechanical but correct — the
  Rust compiler enforces this is the complete set, so the diff confirms every
  consumer was touched.
- `app-server/src/codex_message_processor.rs:241-318,535,2403-2700` — biggest
  semantic chunk. `clear_memory_roots_contents` import moves from
  `codex_core` to `codex_memories_write` (`:317`); `StartThreadWithToolsOptions`
  → `StartThreadOptions` (rename indicates the options shape was generalized
  to include `session_source: Option<...>` at `:2647`); `auth_manager` is
  added to `ListenerTaskContext` at `:537,2406` so the new
  `start_memories_startup_task(thread_manager, auth_manager, thread_id, thread,
  memory_config, session_source)` call at `:2689-2696` has everything it needs.
  The startup-task is fired synchronously (not awaited) right after
  `config_snapshot` is sent — correct ordering: the snapshot tells the IDE the
  thread is running, then the memory startup task fires in the background and
  any `Internal(MemoryConsolidation)` sub-sessions it spawns inherit the live
  trace context.
- `app-server/src/codex_message_processor.rs:3917-3925` — the
  `list_thread_ids` `.collect::<Vec<_>>()` got a turbofish moved to the let
  binding `let mut data: Vec<String> = ...`. Stylistic, not behavioral.

## Nits / what's missing

- `app-server-protocol/schema/json/{ClientRequest.json, codex_app_server_protocol.schemas.json,
  codex_app_server_protocol.v2.schemas.json, v2/RawResponseItemCompletedNotification.json,
  v2/ThreadResumeParams.json}` — every regenerated JSON file lost its trailing
  newline (`\ No newline at end of file`). This is a `ts-rs`-generator output
  glitch (or local CRLF/EOL setting). It's not breaking but it does pollute
  every future regeneration with a one-byte diff. Worth adding `printWidth` /
  `endOfLine` config to the generator script or a post-step `printf "\n" >>`.
- `From<CoreSessionSource> for SessionSource` swallowing `Internal(_)` to
  `Unknown` should emit a `tracing::debug!(?internal_source, "internal session
  source not surfaced to app-server clients")` so the lookup-by-trace path
  stays discoverable.
- `start_memories_startup_task` is called with `Arc::clone(&memory_config)`
  built one block earlier from `config.clone()`. The `Arc<Config>` indirection
  is right (the task lives past the function frame) but the `config.clone()`
  → `Arc::new(...)` → `Arc::clone(...)` chain is a needless allocation for a
  config that's already `Arc`-shared upstream. If `config` is an `Arc<Config>`
  upstream, just clone the `Arc`; if not, this is fine.
- PR body says "This is temporary and it should move at the client level as
  a follow-up" — the `start_memories_startup_task` indirection at app-server
  is the temporary scaffold. Worth a `// TODO(memories-split): move to client`
  comment at the call site so the next engineer doesn't bake assumptions
  about app-server owning this.
- No new test added in this PR diff for the `Internal(MemoryConsolidation)`
  source kind being correctly elided from analytics; the
  `analytics/src/reducer.rs` change is exercised by existing tests but the
  new variant goes through the `(None, None)` arm and a regression that
  reorders the match arms could silently start emitting metrics for
  memory-consolidation sub-sessions. One unit test pinning
  `ThreadMetadataState::from(SessionSource::Internal(_))` returns
  `(None, None)` would close that gap.

## Verdict

**merge-after-nits** — the dep-graph reversal is correctly executed across
the workspace (compiler-enforced exhaustive matches make this visible), the
new `Internal` source kind is appropriately scoped (analytics-elision +
app-server-elision), and the startup-task indirection is the right
temporary-scaffold shape for a multi-PR split. Nits are: re-add trailing
newlines to the regenerated JSON schemas (silence one-byte-per-regeneration
diff noise), debug-log the `Internal → Unknown` conversion in the protocol
shim, mark the `start_memories_startup_task` indirection as `TODO(memories-split)`,
and pin the analytics-elision invariant with a one-line test.
