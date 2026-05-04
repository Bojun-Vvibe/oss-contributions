# openai/codex PR #20910 — frodex: add watchdog runtime handles

- **PR:** https://github.com/openai/codex/pull/20910
- **Author:** friel-openai
- **Head SHA:** `692b62e2` (full: `692b62e278a66d9d9f0e81c02db55a701d38dd56`)
- **State:** OPEN
- **Files touched (selected):**
  - `codex-rs/core/src/agent/watchdog.rs` (+645 / -0) — new module
  - `codex-rs/core/src/agent/control.rs` (+206 / -5) — wire `WatchdogManager` into `AgentControl`
  - `codex-rs/core/src/agent/control_tests.rs` (+416 / -3)
  - `codex-rs/config/src/config_toml.rs` (+5 / -0) — `watchdog_interval_s`
  - `codex-rs/core/config.schema.json` (+12 / -0)
  - `codex-rs/core/src/agent/role.rs` (+27 / -1)

## Verdict

**needs-discussion**

## Specific refs

- `codex-rs/core/src/agent/control.rs:131-140` — new `is_watchdog_helper_source` matcher keys off `SubAgentSource::ThreadSpawn { agent_role: Some("watchdog"), .. }`. Stringly-typed agent role; one typo and the watchdog helper-fork path silently breaks. Should be a typed enum or at minimum a `const WATCHDOG_ROLE: &str = "watchdog";` reused everywhere.
- `codex-rs/core/src/agent/control.rs:155-180` — `AgentControl::new` now eagerly calls `WatchdogManager::new(...).start()`. That means *every* session creates a watchdog manager + spawns its background task, even sessions that never opt in via `watchdog_interval_s`. The old `Default` was zero-cost; this isn't.
- `codex-rs/config/src/config_toml.rs:368-372` — `watchdog_interval_s: Option<i64>` with `#[schemars(range(min = 1))]`. Fine, but `i64` for seconds is overkill — `u32` would be more honest and would serialize identically in TOML. Also no `max` cap, so `watchdog_interval_s = 9223372036854775807` parses cleanly.
- `codex-rs/core/src/agent/watchdog.rs:1-645` — large new module. Did not exhaustively read; the included `control_tests.rs` (+416 lines) is encouraging.
- `codex-rs/config/src/config_toml.rs:670-672` — `#[serde(deny_unknown_fields)]` newly added on `AgentRoleToml`. This is a backwards-incompatible parse change for anyone who has extra/legacy keys in their `[agents.<role>]` block. Should be called out in the PR description.

## Rationale

The "watchdog as a singleton role" framing is reasonable and the test footprint (~400 LOC of unit tests) is the right shape for landing a 645-line behavior module. Three things keep this from being merge-ready:

1. The `serde(deny_unknown_fields)` flip on `AgentRoleToml` is a behavior change unrelated to watchdogs and will surface as parse errors in user configs that previously silently ignored extra keys. Either split it out or document a migration path.
2. The "always-on watchdog manager per session" pattern needs an opt-out (e.g., skip `watchdogs.start()` when `watchdog_interval_s` is None and no role declares `agent_type = "watchdog"`).
3. The stringly-typed `"watchdog"` role check should be promoted to a typed constant.

Code quality is otherwise strong; just wants these tightened before merge.
