---
pr-url: https://github.com/sst/opencode/pull/25158
sha: 4f5af93e44ab
verdict: merge-after-nits
---

# Add SyncEvent service

Promotes the prior module-level `SyncEvent.replay(...)` / `SyncEvent.run(...)` free functions into a proper Effect service (`SyncEvent.Service`) and threads it through the workspace control-plane and session/share/revert paths. The keystone shape change is at `packages/opencode/src/control-plane/workspace.ts:308-358` where the old fire-and-forget `Effect.sync(() => WorkspaceContext.provide({ ..., fn: () => { for (const event of events) SyncEvent.replay(...) } }))` block becomes an `Effect.promise(async () => await WorkspaceContext.provide({ ..., async fn() { await Effect.runPromise(Effect.forEach(events, e => sync.replay(...), { discard: true })) } }))`. That is a load-bearing correctness improvement: replay errors are now surfaced as Effect failures rather than swallowed inside `Effect.sync`, and the SSE-listener arm at `:367-398` now wraps each `sync.replay(payload.syncEvent)` in `Effect.catchCause` that logs and returns a `failed` flag so a single malformed event no longer kills the SSE pump.

Nits: (1) the `Effect.runPromise` inside `WorkspaceContext.provide`'s `async fn` constructs a fresh runtime per replay batch — for large historical replays this is per-batch overhead the prior synchronous form did not have; (2) the `if (failed) return` short-circuits the downstream `GlobalBus.emit` for the same event, which is the right policy but is not explicitly tested.

## what I learned
Lifting a module-level free-function API into a service is a one-time refactor that pays compounding interest: every consumer can now be tested with a fake `SyncEvent.Service` layer, errors flow through the Effect cause tree instead of `try/catch` islands, and the previously-implicit "replay must be sync because we're inside `Effect.sync`" constraint becomes an explicit `Effect`-typed signature.
