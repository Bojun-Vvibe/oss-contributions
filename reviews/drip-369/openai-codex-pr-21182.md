# Review: openai/codex PR #21182

- **Title:** Move installation ID resolution out of core startup
- **Author:** jif-oai
- **Head SHA:** `e018a65e22a9e33cb0601e408f0cc88fd6fff5bb`
- **Files touched:** 20
- **Lines:** +203 / -51

## Summary

Lifts `installation_id` resolution out of the inner thread-manager
construction path and threads it explicitly through
`MessageProcessorArgs`. Concretely:

- `codex-rs/app-server/src/in_process.rs`: `start_uninitialized` is
  now `async` and returns `IoResult`; calls
  `resolve_installation_id(&args.config.codex_home).await?` before
  spawning the runtime task and forwards the value into
  `MessageProcessor` (line ~430).
- `codex-rs/app-server/src/lib.rs:755-778`: same resolve call inside
  `run_main_with_transport_options` before constructing the
  outgoing-message sender.
- `codex-rs/app-server/src/message_processor.rs:258-301`: new
  `installation_id: String` field on `MessageProcessorArgs` and
  destructured into `ThreadManager::new(...)` at line 301.
- Tests updated in `message_processor_tracing_tests.rs` to populate
  the new field.

The point is to have a single early failure path for installation-ID
resolution rather than have it deep inside the core init that runs on
every thread spawn.

## Notes

- `start_uninitialized` going `async` propagates correctly: callers
  in `start` are already in async context and now `.await?` it.
- `resolve_installation_id` is awaited *before* `tokio::spawn`, so a
  failed resolution is surfaced via `IoResult` to the caller instead
  of silently failing inside the runtime task — this is the right
  shape.
- Threading `installation_id: String` through `MessageProcessorArgs`
  is a minor API churn; downstream callers are updated in the same
  PR, including the tracing tests.
- One concern: the value is passed by owned `String`. If
  `MessageProcessor::new` only needs a borrow during construction,
  `Arc<str>` would be cheaper to clone across the multiple consumer
  sites, but with a single consumer per process this is fine.

## Verdict

**merge-after-nits** — consider documenting in the PR description
that the change is a behavior-preserving lift (no change to the
resolved ID semantics, only the call site). Otherwise ready.
