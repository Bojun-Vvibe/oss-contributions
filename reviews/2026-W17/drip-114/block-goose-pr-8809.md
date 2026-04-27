# block/goose PR #8809 — feat(githubcopilot): tag agent-initiated requests with X-Initiator header

- **PR**: https://github.com/block/goose/pull/8809
- **HEAD**: from PR diff body
- **Touch**: 1 source file (`crates/goose/src/providers/githubcopilot.rs`),
  +~85/−~10 with two new tests

## Context

The github-copilot upstream API distinguishes "user-initiated" vs.
"agent-initiated" requests via an `X-Initiator: user|agent` header,
which it uses for billing-tier accounting (agent-initiated calls bill
against a different bucket). goose was sending every request without
this header, which the upstream defaults to `user` — silently
mis-billing every compaction call, every title-generation call, and
every tool-followup call as if the user typed it.

## Diff

`providers/githubcopilot.rs:255-261` adds the header at the
single `post()` callsite:

```rust
let initiator = if is_user_initiated { "user" } else { "agent" };
headers.insert("X-Initiator", initiator.parse().unwrap());
```

The decision of *which* one to send is the interesting part. The PR
uses a `tokio::task_local!` flag `IS_AGENT_CALL` (declared at `:18-21`
with the comment "Task-local so complete() and stream() can't race on
the same provider instance") and the `complete()` impl at `:430-444`
wraps its delegated call in `IS_AGENT_CALL.scope(true, async { ... })`.
`stream()` at `:454-460` reads it with `try_with` (False default) and
combines with a *content-shape* signal:

```rust
let last_is_tool_response = messages.last().is_some_and(|m| {
    m.content.iter().any(|c| matches!(c, MessageContent::ToolResponse(_)))
});
let is_user_initiated = !is_agent_call && !last_is_tool_response;
```

That's the right two-axis classification: (a) `complete_fast()` paths
(compaction, title generation) set the task-local and tag as agent; (b)
otherwise, the heuristic is "if the last message is a `ToolResponse`,
this is an agent-initiated tool followup, not a fresh user message" —
which matches the user-initiated-iff-fresh-user-input intent.

The two new tests are valuable. `is_agent_call_task_local_scoped` at
`:651-664` pins the scoping behavior (visible inside, absent outside)
— this is the mechanism `complete()` relies on, so a future refactor
that drops the `scope(...)` wrapper would now fail loudly.
`tool_response_variant_matches` at `:668-676` pins that the matched
variant is `MessageContent::ToolResponse` (and not, say, `ToolResult`
or `ToolReply` after a rename), again catching a refactor that would
silently flip every tool-followup back to `user`.

## Risks / nits

- `task_local` lives only inside the `tokio::task_local!` scope; if a
  future caller invokes `complete()` and somehow calls `stream()`
  *outside* the `.scope(...)` (e.g., from a `tokio::spawn`'d future
  that doesn't inherit the task-local), the agent-tag would silently
  drop. A future test along the lines of "spawn a task inside
  IS_AGENT_CALL.scope and verify the spawned task does NOT see the
  flag" would document this boundary explicitly.
- `is_user_initiated = !is_agent_call && !last_is_tool_response` —
  consider the case where a user manually pastes a tool response (e.g.,
  via copy-paste of an MCP tool result into the prompt) as the last
  message before re-running. The heuristic would tag it as agent. This
  is rare but the comment should call it out so a future "but the user
  *did* type this" bug report has a paper trail.
- `headers.insert("X-Initiator", initiator.parse().unwrap())` — the
  `.unwrap()` on a literal `"user"` / `"agent"` cannot fail, but a
  `HeaderValue::from_static(initiator)` would be both faster and
  panic-free by construction.
- The PR fixes only the github-copilot provider. If other API-tier-
  charging providers gain a similar header convention later, the
  task-local + content-heuristic pair belongs in a shared base layer
  rather than copy-pasted per provider. Worth a `// TODO: hoist to
  base if a second provider needs this` comment.
- No test exercising the *combination* — i.e., `complete()` calling
  `stream()` and verifying that the resulting `post()` call carries
  `X-Initiator: agent`. The two existing tests pin the building blocks
  but not the wiring.

## Verdict

Verdict: merge-after-nits
