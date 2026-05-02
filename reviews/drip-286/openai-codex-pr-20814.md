---
repo: openai/codex
pr: 20814
head_sha: 987edd053c6391e35f2e85cdd036f838e7c4e0a3
title: "[DO NOT MERGE] Prototype: extract goals into app-server runtime extension"
verdict: needs-discussion
reviewed_at: 2026-05-03
---

# Review: openai/codex#20814 — `[DO NOT MERGE] Prototype: extract goals into app-server runtime extension`

**Head SHA:** `987edd053c6391e35f2e85cdd036f838e7c4e0a3`
**Stat:** +2911 / −3165. Author: etraut-openai. State: DRAFT,
explicitly marked `[DO NOT MERGE]`.

This is a prototype PR by the author's explicit declaration. Reviewing
it as an architectural proposal, not a merge candidate.

## What it changes

Lifts goal ownership out of `codex-core::Session` and into
`codex-rs/app-server`, replacing the core-owned policy surface with a
generic `SessionRuntimeExtension` trait that hosts can install:

```rust
pub trait SessionRuntimeExtension: Send + Sync {
    fn tool_specs(&self, context: SessionToolSpecContext) -> Vec<ToolSpec>;
    fn handle_tool_call(...) -> BoxFuture<'_, Result<SessionToolOutput, SessionToolError>>;
    fn on_event(...) -> BoxFuture<'_, anyhow::Result<()>>;
    fn next_idle_background_turn(...) -> BoxFuture<'_, anyhow::Result<Option<SessionBackgroundTurn>>>;
}
```

App-server side adds `goal_runtime` module
(`app-server/src/goal_runtime/`); `CodexMessageProcessor` gains
`goal_runtime: Option<Arc<GoalRuntime>>` (line 557), wired up in
`new()` at lines 875–882 conditional on `Feature::Goals`. Routing is
renamed from `thread_goal_handlers` → `goal_handlers` and the
`thread_goal_event` local is renamed to `goal_event`
(`bespoke_event_handling.rs:1238–1243`).

The continuation hook is restructured as a *provider* API
(`next_idle_background_turn`) rather than the previous lifecycle event
(`MaybeContinueIfIdle`). Core calls into the extension at idle points
and the extension may return hidden input; core retains race-check and
turn-start ownership. Tools are merged into the normal `ToolRouter`
alongside core/MCP/dynamic/deferred tools at turn setup.

Public app-server `thread/goal/*` API shape is preserved per PR
description; only internal routing and ownership moved. Goal
persistence stays in `codex-state`.

## Assessment

### What works

- The boxed-future trait shape (vs `async_trait`) is the right call for
  a public-ish core extension surface — it avoids the Box<dyn> +
  trait-object lifetime gymnastics that `async_trait` papers over and
  keeps the "host can implement this" surface explicit.
- The `next_idle_background_turn` provider redesign described in
  "Latest Prototype Refinement" is the right shape. Letting the
  extension return *hidden input* and keeping race-check + turn-start
  in core preserves the invariant that there's exactly one place
  responsible for "is it safe to start a turn right now". The previous
  `MaybeContinueIfIdle` event + extension-side `start_turn` call was
  reversing that responsibility and would have been a footgun.
- `ThreadUnloadContext` struct (lines 562–569) cleans up an existing
  6-arg `unload_thread_without_subscribers` signature that this PR
  would otherwise have made 7-arg. Good incidental cleanup.
- `goal_runtime.clear_thread_state(thread_id)` at line ~6027 wired into
  the unload path means goal state lifetime tracks thread lifetime.

### What needs discussion

- **Two simultaneous extensions**: the trait is `Option<Arc<dyn
  SessionRuntimeExtension>>` — explicitly only one extension per
  session. If the next product policy (e.g. a future "remote
  compaction" extension or the existing `feature config hints`
  surface) also wants to be a runtime extension, this trait shape
  forces it to either compose at the *host* level (app-server
  multiplexes multiple extensions onto one trait impl) or to redesign
  the extension surface. Worth deciding now whether to make the slot
  `Vec<Arc<dyn ...>>` with a documented merge order, or whether
  app-server-side composition is the intended pattern.
- **Tool spec naming collisions**: `SessionToolSpecContext { mode,
  ephemeral }` returns `Vec<ToolSpec>` from the extension; these merge
  into the same `ToolRouter` as core/MCP/dynamic/deferred. The PR
  description doesn't say what happens on name collision. With the
  extension surface this is not "user wrote two MCP servers with the
  same tool name" (which has a defined error path) but "core ships a
  tool, extension also tries to ship a tool with the same name" —
  needs an explicit resolution policy (extension wins? core wins?
  error at install?).
- **Feature flag coupling**: extension is installed only when
  `Feature::Goals` is enabled (lines 875–882). That works for the
  goals migration but means the extension surface itself is gated by a
  feature flag, not by "is there an extension to install". For a
  generic extension surface, the feature gate should probably move
  inside the goal-runtime install (i.e. always allow extension
  install, but the goal-runtime code itself decides whether to
  register based on `Feature::Goals`).
- **Renaming `thread_goal_*` → `goal_*`**: cosmetic but touches every
  call site. Worth confirming with reviewers that the shorter name
  doesn't conflict with any other "goal" concept once goals stop being
  thread-bound at the persistence layer.

### Risk surface

- 2911 added / 3165 deleted is a near-1:1 lift-and-shift, but the diff
  truncated before showing the actual `goal_runtime/` module, the new
  `session_extension.rs`, and the deleted core goal files. Net −254
  is consistent with "prototype, not yet fully tested" — production
  version will likely add lines.
- "Public app-server `thread/goal/*` API shape unchanged" claim is the
  most important invariant for downstream clients; needs a contract
  test (or a doc-test) that pins the JSON-RPC surface.

## Verdict

**`needs-discussion`** — author asked for architectural review only,
which is appropriate for a 6k-line refactor of core ownership. The
shape is sound; the open questions above (extension multiplicity,
tool-name collision policy, feature-flag scope) need answers in PR
discussion before this graduates from `[DO NOT MERGE]`. Not blocking
on the prototype; this verdict reflects the conversation the author
explicitly invited.
