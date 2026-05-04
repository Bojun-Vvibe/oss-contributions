# openai/codex#20912 — synchronize agent control tools

- **Head SHA:** `84a34f29926d77408859f1fab92c824dba3a5ad1`
- **Author:** internal contributor
- **Size:** +1615 / −100, 19 files (codex-rs/core agent + tools)

## Summary

Keeps the agent-control tool surface aligned across root agents, forked agents, and watchdog helper forks. Motivation: stable model-visible tool list preserves prompt-cache sharing across agent forks. Restores watchdog helper control tools as eager tools, preserves parent-messaging support needed by watchdogs, all without changing the semantics of existing tools.

## Specific citations

- `core/src/agent/control.rs:91-101` — new `WatchdogParentCompactionResult` enum with three variants: `NotWatchdogHelper`, `ParentBusy { parent_thread_id }`, and `Submitted { parent_thread_id, submission_id }`. Clean tagged result type, much better than a multi-bool return.
- `core/src/agent/control.rs:61-62` — two new constants `WATCHDOG_BOOT_TOOL_SEARCH_CALL_ID` and `WATCHDOG_BOOT_LIST_AGENTS_CALL_ID` with `synthetic_*` prefixes. Worth verifying nothing in the rollout-replay path treats `synthetic_*` as a reserved namespace; if not, document it as such.
- `core/src/agent/control.rs:174-209` — `synthetic_watchdog_tool_search_items()` constructs synthetic `RolloutItem::ResponseItem(ToolSearchCall)` + `ToolSearchOutput` pairs from `create_compact_parent_context_tool` + `create_watchdog_close_self_tool` + `create_watchdog_snooze_tool`. Bails with empty `Vec` if `serde_json::to_value(namespace)` fails (`:179-181`) — silent failure mode; a `tracing::warn!` would help if this ever trips in production.
- `core/src/agent/control.rs:167-172` — `unix_timestamp_seconds()` returns `0` on `duration_since(UNIX_EPOCH)` error via `.unwrap_or_default()`. System clock pre-1970 is exotic but the silent zero could be misleading in logs; consider `tracing::warn!` on the err arm.
- `core/src/agent/control_tests.rs`, `core/src/tools/handlers/multi_agents_tests.rs` — both updated. Couldn't audit the full 1500+ diff in this slice. The validation command `CODEX_SKIP_VENDORED_BWRAP=1 cargo test -p codex-core thread_rollout_truncation` covers the rollout-truncation interaction, which is the highest-risk path given the synthetic ResponseItems.
- `core/src/tools/spec.rs`, `tools/src/agent_tool.rs` — modify the tool catalog. These are the contact surface with prompt-cache sharing; changes here are the load-bearing part of the PR's stated motivation.

## Verdict: `needs-discussion`

The mechanism (synthetic tool-search rollout items to ensure consistent tool visibility for prompt-cache stability) is reasonable but introduces non-trivial new state. Concerns to discuss before merge:

1. **Synthetic rollout items semantics.** Inserting `ToolSearchCall` / `ToolSearchOutput` rollout items that the model never actually executed creates a divergence between the rollout-as-recording and the rollout-as-replay. Replaying these synthetics on a fresh thread will *also* synthesize them, but if the synthesis logic ever changes, prompt-cache invalidation could cascade unexpectedly. Worth documenting that the synthetic prefix is a stable contract.
2. **1615 added LOC across 19 files in one PR is a lot for "synchronize agent control tools"**. Specifically, `multi_agents_v2/{followup_task,message_tool,send_message,spawn}.rs` and the `watchdog_self_close.rs` / `compact_parent_context.rs` handlers are all touched. The PR description doesn't enumerate which behaviors changed in each. Recommend either (a) splitting into a synthetic-rollout-items PR, a tools-spec sync PR, and a watchdog helper PR, or (b) expanding the description to cover what each file's change is doing.
3. **`unwrap_or_default()` on `duration_since`** returning 0 silently — minor, but consider logging.
4. **No assertion that the synthetic items are skipped from analytics / token-counting.** If they're counted toward token budgets they'll inflate numbers; verify in `multi_agents_tests.rs`.

The change appears intentional and the tests look thorough at the file-name level; it's the scope and the synthetic-replay contract that need explicit sign-off.
