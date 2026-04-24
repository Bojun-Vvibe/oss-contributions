# openai/codex PR #19452 — Stabilize plugin MCP fixture tests

- **Repo:** openai/codex
- **PR:** [#19452](https://github.com/openai/codex/pull/19452)
- **Head SHA:** `11feeaa00f719d52787984ebbf221e2b581d38b7`
- **Author:** dylan-hurd-oai (Dylan Hurd)
- **Size:** +41 / −35 across 1 file
- **Reviewer:** Bojun (drip-26)

## Summary

Fixes a flake in two tests in `codex-rs/core/tests/suite/plugins.rs`:
`explicit_plugin_mentions_inject_plugin_guidance` and
`plugin_mcp_tools_are_listed`. Both depended on the sample MCP server
being fully started before the test asserted on plugin guidance / tool
inventory. `plugin_mcp_tools_are_listed` already had a local readiness
wait inlined; the other test had none and just submitted the user turn
immediately after constructing the session. On a slow CI runner the
prompt could be built before MCP startup completed, and the assertion
would then flake against an empty plugin inventory.

The fix extracts the readiness check into a shared
`wait_for_sample_mcp_ready(&codex)` helper and calls it from both
tests before they exercise the fixture.

## Key changes (single file)

### `codex-rs/core/tests/suite/plugins.rs:153–193`

New helper added:

```rust
async fn wait_for_sample_mcp_ready(codex: &codex_core::CodexThread) -> Result<()> {
    let startup_event = wait_for_event_with_timeout(
        codex,
        |ev| match ev {
            EventMsg::McpStartupComplete(summary) => {
                summary.ready.iter().any(|server| server == "sample")
                    || summary.failed.iter().any(|failure| failure.server == "sample")
                    || summary.cancelled.iter().any(|server| server == "sample")
            }
            _ => false,
        },
        Duration::from_secs(70),
    ).await;
    ...
}
```

It waits for an `McpStartupComplete` event mentioning the `sample`
server in any terminal state (`ready` / `failed` / `cancelled`),
then converts `failed` and `cancelled` into a `bail!` so the test
gets a precise failure message instead of a generic timeout.

### `codex-rs/core/tests/suite/plugins.rs:307` (call site)

`explicit_plugin_mentions_inject_plugin_guidance` now invokes
`wait_for_sample_mcp_ready(&codex).await?` immediately after building
the codex thread and before the first `Op::UserInput`. This is the
actual fix for the flake — the previous version had no readiness
wait at all on this path.

### `codex-rs/core/tests/suite/plugins.rs:456–459`

`plugin_mcp_tools_are_listed` deletes its inlined readiness check
(35 lines) and replaces it with a single `wait_for_sample_mcp_ready`
call. Pure refactor; behaviour is identical to the inlined version
because the helper preserves the same predicate, timeout, and
failure semantics.

## What's good

- The flake diagnosis matches the linked CI runs (24909500958,
  24908076251, 24906197645, 24898949647). Same test family, same
  symptom, addressed at the actual root cause (missing readiness
  signal) rather than by adding sleeps or bumping timeouts.
- The 70-second timeout is preserved verbatim from the original
  inlined version, so this PR is not also a stealth timeout bump.
- Distinguishing `failed` / `cancelled` from `ready` and converting
  the former into `bail!` with the underlying error string keeps
  diagnostic quality high — a real MCP-startup regression will
  surface as `plugin MCP server failed to start: <error>` instead
  of a 70s timeout with no detail.
- One helper, two call sites, no behavioural change to the runtime
  code under test. Exactly the right scope for a flake fix.

## Concerns

1. The `unreachable!("event guard guarantees McpStartupComplete")`
   pattern at line 172 is fine in test code, but if anyone later
   adds a second predicate branch in `wait_for_event_with_timeout`
   that *can* match a non-`McpStartupComplete` event, this turns
   into a panic instead of a graceful test failure. Worth a comment
   noting that the predicate is intentionally narrow.

2. The helper is currently file-local. If a future plugin test
   needs the same wait, the next author will probably copy-paste
   it into another `suite/*.rs` file (history rhymes — that's how
   the inlined version got there in the first place). Worth pulling
   into a `tests/common/` module if a third caller appears.

3. Nit: the `assert!` at the end of the helper is functionally
   redundant with the predicate — by the time we get here, the
   predicate already proved one of the three sample-server arms
   was true, and the `failed`/`cancelled` arms `bail!` first. The
   `assert!` only fires if the predicate matched on a summary that
   contained the sample server in `ready` (which we'd want to
   succeed) but somehow doesn't anymore. It's belt-and-braces and
   harmless.

## Risk

Very low. Test-file-only change. Pure deduplication plus adding a
missing wait in one test. The CI link in the PR body confirms the
target tests pass.

## Verdict

**merge-as-is**

Right diagnosis, minimal blast radius, no behavioural change to
production code. The two nits above are documentation-only and
not worth blocking on.

## What I learned

The `findLast`-style "match any terminal state for a specific
subject" pattern in test event waiters is a good antidote to
"happy-path-only readiness" flakes. The test only ever waited for
`ready` originally; if startup actually failed, the test would
hang the full timeout instead of failing fast with the underlying
error. Folding `failed` and `cancelled` into the same predicate
and then branching on the variant inside the helper is a clean
way to keep the wait narrow but the diagnostics broad.
