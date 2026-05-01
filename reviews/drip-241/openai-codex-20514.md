# openai/codex#20514 — [codex-analytics] add core item timing production

- **PR**: https://github.com/openai/codex/pull/20514
- **Head SHA**: `42cdca0421e9ef11af8950502eae17b1811b6ba6`
- **Files**: 30 files; `core/src/event_mapping.rs`, `core/src/mcp_tool_call.rs`, `core/src/session/*`, `core/src/tasks/user_shell.rs`, `core/src/tools/events.rs`, `core/src/tools/handlers/*` (multi-agents v1 + v2 + dynamic + mcp_resource), `protocol/src/items.rs` (+53), `protocol/src/protocol.rs` (+133). Stack base PR for #20515.
- **Verdict**: **merge-after-nits**

## Context

Producer-side foundation for codex-analytics item timing. Adds `started_at_ms: Option<i64>`, `completed_at_ms: Option<i64>`, `duration_ms: Option<i64>` fields to the core protocol model for tool families that already emit begin/end events: web search, image generation, MCP, dynamic tools, command execution, patch application, collaboration tools. Captures and propagates timestamps across every core producer. Stack ordering is correct — this PR is the foundation that #20515 builds on for the public wire.

## What's right

- **Protocol additions concentrated in two files.** `protocol/src/protocol.rs:+133` and `protocol/src/items.rs:+53` carry the bulk of the schema. The `Option<i64>` shape is correct (timing isn't always known, e.g., for items reconstructed from old rollouts).
- **Producer-side capture is thorough.** Every event-emitting code path gets `started_at_ms`/`completed_at_ms`/`duration_ms` plumbed through:
  - `core/src/mcp_tool_call.rs:+19` — captures wall-clock around MCP tool invocation.
  - `core/src/tasks/user_shell.rs:+12` — covers the user-shell command path (which has its own lifecycle vs. the agent-shell path).
  - `core/src/tools/handlers/multi_agents/{close_agent,resume_agent,send_input,spawn,wait}.rs` (+10/+6/+6/+6/+10) and `multi_agents_v2/{close_agent,message_tool,spawn,wait}.rs` (+10/+6/+6/+6) — both v1 and v2 multi-agent paths covered symmetrically.
  - `core/src/tools/handlers/multi_agents_common.rs:+27` — common helper that v1 and v2 likely share.
  - `core/src/tools/handlers/dynamic.rs:+8`, `mcp_resource.rs:+43/-3` — dynamic tool and MCP resource handlers.
- **Repeating `started_at_ms` on end events is deliberate.** PR body: "Repeating `started_at_ms` on end events lets reducers consume a complete interval from the completion event itself instead of retaining request-time state, which keeps the timing data colocated in one place." This is the right discipline — stateless reducers are cheaper to reason about and don't lose state on restart.
- **Test additions cover the right surfaces.** `core/tests/suite/items.rs:+39` (item-level emission), `core/tests/suite/rmcp_client.rs:+5` (MCP), `core/tests/suite/user_shell_cmd.rs:+11/-1` (user shell with interrupt), plus `core/src/event_mapping_tests.rs:+12` (event-mapping round-trip), `core/src/session/tests.rs:+2` and `core/src/session/session.rs:+4`. PR body's verification: `cargo test -p codex-core web_search_item_is_emitted` and `cargo test -p codex-protocol`.
- **`turn_timing.rs` (+1/-1)** — minimal touch, suggests turn-level timing already existed and this PR slots into the existing infrastructure rather than rebuilding it.
- **`app-server-protocol/src/protocol/thread_history.rs` (+35)** — the producer-side thread_history change is small and limited to test arms with `started_at_ms: None, completed_at_ms: None, duration_ms: None` mechanical insertions (visible in the diff at `:1426`, `:1813`, `:1829`, `:2068`, `:2322`, `:2415`, `:2734-2737`, `:2819`, `:2887`, etc.). Mechanical sweep, low risk.

## Risks / nits

- **Mechanical `started_at_ms: None, completed_at_ms: None, duration_ms: None` insertions across ~20 test construction sites.** Easy to miss one. The compile catches missing-field errors (since the protocol structs are non-`#[serde(default)]` for these fields), but a test that *intentionally* should exercise the timing path could pass with `None,None,None` and look correct. Recommend at least one test that asserts non-None timing values are correctly propagated end-to-end (most tests appear to use `None` placeholders).
- **`mcp_tool_call.rs:+19` captures `started_at_ms` at MCP-call entry but `completed_at_ms` should be captured at `?` propagation boundaries too.** If an MCP call errors out mid-flight, does the error path still record `completed_at_ms`? Spot-check the diff suggests yes (the timing capture is around the `Result` boundary), but worth a regression test for the error-path completion timestamp.
- **No PR-body discussion of clock source.** Is `started_at_ms` from `SystemTime::now().duration_since(UNIX_EPOCH)` (wall-clock, can go backwards on NTP adjust) or from `Instant::now()` (monotonic, can't be expressed as Unix epoch)? The schema name "Ms" + Unix-epoch description in #20515 implies wall-clock. Recommend a comment near the capture sites pinning that wall-clock is intentional (so consumers know `completedAtMs < startedAtMs` is theoretically possible across an NTP step).
- **`core/src/tools/events.rs:+20`** changes the events module shape; want a doc comment on the new fields explaining when each is populated (e.g., `started_at_ms` set at handler entry, `completed_at_ms` set at handler exit, `duration_ms` derived as `completed - started`).
- **PR depends on its stack siblings.** This is the producer foundation; #20515 is the public-wire layer. Worth flagging in CI/merge-queue that landing this without #20515 would mean core records timing that no app-server consumer can read.

## Verdict

**merge-after-nits.** Solid producer-side foundation laid symmetrically across every core item-emitting handler with stateless-reducer-friendly "repeat `started_at_ms` on end events" discipline. The `Option<i64>` shape correctly leaves room for unknown timing. Strongest nits: at least one positive test pinning real timing values flow through (vs the `None,None,None` mechanical inserts), explicit error-path completion timestamp coverage, and a comment pinning wall-clock vs monotonic clock source so consumers can reason about the `completedAtMs < startedAtMs` edge case.
