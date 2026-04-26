# PR #8842 — feat: lifecycle hooks system

- **Repo**: block/goose
- **PR**: #8842
- **Head SHA**: `83c0dcfebf305c8843e4263c716aa0475a5ef401`
- **Author**: michaelneale
- **Size**: +550 / -10 across 3 files
- **Verdict**: **needs-discussion**

## Summary

Introduces a lifecycle-hooks system for the Goose agent. New
module `crates/goose/src/hooks.rs` (+444) defines `HookEvent`,
`HookContext`, `HookManager`, and a `HookManager::from_config()`
constructor. The agent gains a `hook_manager` field
(`agent.rs:155`) and emits `BeforeToolCall` / `AfterToolCall`
events around the tool-execution path, with the `BeforeToolCall`
hook able to *block* execution and return a structured error.

## Specific changes

- `crates/goose/src/lib.rs:+1` — module registration.
- `crates/goose/src/agents/agent.rs:155` — `hook_manager:
  crate::hooks::HookManager` field on `Agent`.
- `agent.rs:255` — `HookManager::from_config()` invoked at
  agent construction. Means hooks are loaded once per agent;
  no live reload yet.
- `agent.rs:286-303` — public `emit_hook(event, session_id,
  working_dir)` helper for CLI/server-driven session events
  (e.g. session-start, session-end). Gated by
  `has_hooks(event)` to avoid building the context object when
  no hook is registered. Sensible perf shape.
- `agent.rs:594-625` — `BeforeToolCall` emit before tool
  dispatch. Builds `HookContext { event, session_id, tool_name,
  tool_input, tool_result: None, message: None, working_dir
  }`. If `decision.block`, returns
  `ErrorData::new(ErrorCode::INVALID_REQUEST, "Tool call
  blocked by hook: <reason>", None)`. The error category is the
  one nit: this is a *policy decision*, not an
  `INVALID_REQUEST`. Should probably be its own code (e.g.
  `POLICY_BLOCKED`) so callers can distinguish "user wrote a
  bad tool call" from "your hook said no."
- `agent.rs:692-712` — `AfterToolCall` emit. Note the comment
  "fire-and-forget, no blocking" — this is an explicit design
  choice: post-tool hooks can observe but not mutate. **But**
  the diff still passes `tool_result: None` here too. That's
  almost certainly wrong — the whole point of an
  `after_tool_call` hook is access to the result. See
  `risks/2`.
- The non-hook reordering of `use` statements (anyhow, futures,
  tokio::sync) is incidental import-ordering churn. Annoying
  noise but not blocking.

## Risks

1. **Untrusted hook config = arbitrary code path manipulation**.
   The PR text presumably explains where hook commands come
   from, but `HookManager::from_config()` reading from the
   ambient Goose config means a malicious config file (or a
   rogue extension that writes config) can install a hook that
   blocks arbitrary tool calls or exfiltrates tool input. Needs
   a security model section.
2. **`tool_result: None` in `AfterToolCall`** — almost certainly
   a bug. The hook fires *after* `debug!("WAITING_TOOL_END:
   ...")` but doesn't seem to thread the actual result into the
   context. If the goal is observation, the hook is observing
   nothing. (Could be intentional to keep the hook signal
   lightweight — but then the field shouldn't exist on the
   context struct.)
3. **No timeout on hook execution**. A misbehaving hook script
   can hang the agent indefinitely on every tool call. Need at
   minimum a config-driven timeout, ideally with a "hooks fail
   open" default.
4. **Error code choice** — `INVALID_REQUEST` for a hook block
   conflates user error with policy enforcement.
5. The 444-line `hooks.rs` isn't visible in the head-200 cut —
   would want to read it before approving. Likely contains the
   `HookManager::emit` impl, the `HookDecision` shape, and the
   subprocess execution model.

## Verdict

`needs-discussion` — the feature is reasonable and well-scoped
on the call-site side, but four design questions need
resolving before merge: (a) trust model for hook config,
(b) whether `tool_result` should actually flow into
`AfterToolCall`, (c) execution timeout, (d) error-code
taxonomy for blocked calls.

## What I learned

Lifecycle hooks are one of those features that look small at
the call site (10 lines around each tool call) and become
sprawling at the policy edge (who installs hooks? what can
they see? what can they block?). The right move on a PR like
this is to nail the security model in the design doc *before*
the call sites land, because once the API ships you're stuck
with whatever the hook context exposes.
