---
pr_number: 19169
repo: openai/codex
head_sha: 5c332974096833b879c1cfdb51cc5f1b057405a4
verdict: merge-as-is
date: 2026-04-25
---

# openai/codex#19169 — stabilize `plugin_mcp_tools_are_listed`

**What changed.** +47 / −37 in one file (`core/tests/suite/plugins.rs`). Two coupled fixes:

1. **Fixture lifetime.** Three helpers (`build_plugin_test_codex` line 100, `build_analytics_plugin_test_codex` line 110, `build_apps_enabled_plugin_test_codex` line 124) previously returned `Arc<codex_core::CodexThread>` after destructuring `.codex` out of `TestCodex`. Now they return the full `TestCodex` fixture, and callers do `let codex = Arc::clone(&fixture.codex)` (lines 184, 265, 349, 416). The fixture owns a `TempDir` cwd guard; previously, that guard could drop after the helper returned but *before* local stdio MCP startup completed.

2. **Replace polling with event-driven assert.** Old code (lines 414–443 of the prior file) was a 30s deadline loop submitting `Op::ListMcpTools` every 200 ms, asserting `tool_list.tools` contained `mcp__sample__echo` and `mcp__sample__image`. New code (lines 416–434) waits exactly once for `EventMsg::McpStartupComplete` (20s timeout) and asserts `startup.ready.iter().any(|server| server == "sample")`, then issues a single `Op::ListMcpTools` and asserts the two tool keys are present.

**Why it matters.** Body says: "The Linux cargo failure was a nextest timeout in `suite::plugins::plugin_mcp_tools_are_listed`. The test helper kept only `CodexThread` and dropped the surrounding `TestCodex` fixture, which can drop the temporary cwd guard before local stdio MCP startup finishes. That became easier to hit after the local stdio cwd fallback path landed." This is the textbook flaky-test fix: replace polling-with-timeout with a deterministic readiness signal that already exists.

**Concerns.**
1. **Polling loop is gone, but the new code asserts `startup.ready.iter().any(|server| server == "sample")`** with `failed` and `cancelled` printed in the panic message (line ~430). Good. However if `McpStartupComplete` fires *before* the `wait_for_event_with_timeout` registration (race against fast servers), the test could still hang. Look at `wait_for_event` semantics: if it observes only events arriving *after* registration, this is a real race. Most async event helpers in this repo buffer pre-registration events for known event types, but worth confirming for `McpStartupComplete` specifically.
2. **20-second startup timeout vs. the prior 30-second polling deadline** is a tightening. On a slow CI runner with stdio MCP server cold-start (rmcp_test_server_bin spawn + handshake), 20s is comfortable but not generous. If the test starts to flake again on Linux, bump to 30s before reverting.
3. **The other three helpers also flipped to returning `TestCodex`** even though only `plugin_mcp_tools_are_listed` had the documented flake. That's the right call — fixture-lifetime bugs are global — but it means callers in other tests now hold onto a `TempDir` for a longer scope, increasing the parallel-test disk footprint marginally. Negligible.
4. **`use core_test_support::test_codex::TestCodex;`** added at line 21. Confirm `TestCodex` is `pub` — the diff suggests yes (other helpers reference it), but worth `cargo check`ing.
5. **`Validation` section says "I did not run local cargo/nextest validation per the Codex repo workflow; CI should exercise the target matrix."** That's fine for a test-only fix, but the PR is *itself* a CI-stability fix — landing it without a green CI run on the target matrix means the fix gets validated in the wild. Wait for CI green before merge.

Surgical, well-justified flaky-test fix. Land after a CI green run.
