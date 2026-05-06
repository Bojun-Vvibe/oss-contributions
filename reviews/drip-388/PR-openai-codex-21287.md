# openai/codex#21287 — Move skills watcher to app-server

- PR: https://github.com/openai/codex/pull/21287
- Head SHA: `6fcdafdc150a2a1581e733e4dd80ad45a8d46c10`
- Size: +198/-417 (net -219, classic extraction)
- Verdict: **merge-after-nits** (man)

## Shape

Hoists the skills file watcher out of `codex-core` and into the `codex-app-server`
crate as a new `mod skills_watcher;` module, registered at
`codex-rs/app-server/src/lib.rs:94`. The watcher is constructed once in
`MessageProcessor::new` at `message_processor.rs:310`
(`SkillsWatcher::new(thread_manager.skills_manager(), outgoing.clone())`),
threaded as `Arc<SkillsWatcher>` into both `RequestProcessors` (at `:404`) and
`TurnRequestProcessor` (at `:418`). The bespoke event handler that previously
fanned `EventMsg::SkillsUpdateAvailable` into a
`ServerNotification::SkillsChanged` at `bespoke_event_handling.rs:204-210` is
deleted — the watcher now emits directly without round-tripping through the
core event bus, which is the conceptual win.

## Notable observations

- The deleted block at `bespoke_event_handling.rs:204-210` and the removed
  `SkillsChangedNotification` import at `:53` mean any out-of-tree consumer
  that was driving skills-changed via `EventMsg::SkillsUpdateAvailable` from
  core will silently stop receiving notifications. The watcher's direct path
  is correct, but the `EventMsg::SkillsUpdateAvailable` variant either should
  also be deleted from `codex-protocol` in the same PR (so the dead path
  fails-fast) or explicitly documented as "core-emitted, no longer routed by
  app-server".
- Uses `Arc::clone(&skills_watcher)` for both processors at `:404` and `:418`
  (good — no double-construction). The watcher must therefore be `Send + Sync`
  internally; if it owns a `tokio::sync::watch::Sender` that needs `&mut self`
  to send, the implementation will need `Mutex` wrapping.
- This PR is a sibling/sequel to #21290 (file watcher to its own crate) and
  the same reviewer concerns apply: no `Cargo.toml description/repository`
  shown for the new `skills_watcher` module, but since this is in-crate not a
  new workspace member, it's a smaller surface than #21290.

## Concerns / nits

- No test in the diff that exercises the new `SkillsWatcher` path end-to-end
  (write a `SKILL.md`, expect a `ServerNotification::SkillsChanged` on the
  outgoing channel within N ms). The previous core path presumably had one;
  if so, it should be moved alongside the code.
- `EventMsg::SkillsUpdateAvailable` should either be removed from
  `codex-protocol` in this PR (to enforce the dead-path) or get a
  `#[deprecated(note = "Use SkillsWatcher in app-server")]` on the variant.
- `MessageProcessor::new` is now noticeably longer; consider extracting
  `build_skills_watcher(thread_manager, outgoing)` for symmetry with the other
  processors built in this constructor.
