# openai/codex#19474 — Make thread store process-scoped

- PR: https://github.com/openai/codex/pull/19474
- Author: wiltzius-openai (Tom)
- +47 / -48
- Head SHA: `8f6aed7f69a9f3dfe34dc42aed479fda1b41323b`

## Summary

Hoists thread-store construction up to `MessageProcessor` startup so a single
`Arc<dyn ThreadStore>` is shared by `ThreadManager`, `CodexMessageProcessor`,
and every fork path. Removes the per-thread/fork `configured_thread_store(&config)`
calls so that an effective per-turn `Config` cannot quietly switch the persistence
backend (Local vs Remote vs InMemory) mid-session. The previously private
`configured_thread_store` is renamed and re-exported as
`codex_core::thread_store_from_config` so callers (app-server, mcp-server,
prompt_debug, tests) all funnel through the same constructor.

## Specific findings

- `codex-rs/core/src/thread_manager.rs:255` — `pub fn thread_store_from_config(...)`
  is now the single canonical builder; previously this was a private
  `configured_thread_store` duplicated in `app-server/src/codex_message_processor.rs`.
  Good single-source-of-truth move.
- `codex-rs/app-server/src/message_processor.rs:289` (SHA
  `8f6aed7f69a9f3dfe34dc42aed479fda1b41323b`) — `let thread_store =
  thread_store_from_config(config.as_ref());` is built once at
  `MessageProcessor::new` and `Arc::clone`d into both `ThreadManager::new` and the
  `CodexMessageProcessor` args. Process-scoped lifetime is exactly what the PR
  description claims.
- `codex-rs/core/src/thread_manager.rs:368` — the `#[cfg(test)]`-style
  constructor (no `Config` input) hard-codes a `LocalThreadStore` with
  `OPENAI_PROVIDER_ID` and `generate_memories: false`. The inline comment
  explicitly says "Tests that need a non-local process store should construct
  ThreadManager::new with an explicit store." That's fine, but note it changes
  behavior for any existing test that previously relied on
  `experimental_thread_store = Remote` flowing through this constructor — those
  tests will now silently get Local. Worth grepping for `ThreadManager::for_test`
  callers that set a remote store config.
- `codex-rs/app-server/src/codex_message_processor.rs:5004` — the fork path
  drops the `thread_store: &dyn ThreadStore` parameter from
  `read_stored_thread_for_new_fork` and uses `self.thread_store` directly. This
  is the load-bearing behavior change: forks can no longer pick a different
  backend than the parent session. Matches the PR's stated goal.
- `codex-rs/core/src/prompt_debug.rs:53` — `build_prompt_input` now constructs
  its own `thread_store_from_config(&config)` per call. Acceptable for a debug
  helper, but confirms the "process-scoped" guarantee is enforced at
  `MessageProcessor`, not in `ThreadManager` itself.

## Verdict

`merge-after-nits`

## Rationale

Mechanical, well-scoped refactor that closes a real footgun (per-fork backend
swap). The only thing I'd ask before merge is a one-line doc-comment on
`ThreadManager::new` clarifying that callers MUST pass the same `Arc<dyn
ThreadStore>` for the lifetime of the process, and a quick audit of the
`#[cfg(test)]` constructor at `thread_manager.rs:368` for any pre-existing test
that depended on a Remote store via config — those now silently fall back to
Local. The `mcp-server/src/message_processor.rs` 2-line hookup is included so
the mcp-server stays in lockstep, which is the right call.
