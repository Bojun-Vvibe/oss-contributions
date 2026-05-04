# openai/codex PR #20914 — frodex: restore fork command and debug hooks

- **PR:** https://github.com/openai/codex/pull/20914
- **Author:** friel-openai
- **Head SHA:** `4999ef03` (full: `4999ef03cb0a3b2075121ee28d5e8244e33362fc`)
- **State:** OPEN
- **Files touched (selected):**
  - `codex-rs/core/src/agent/control.rs` (+9 / -7)
  - `codex-rs/core/src/agent/control_tests.rs` (+7 / -2)
  - `codex-rs/core/src/session/session.rs` (+12 / -1)
  - `codex-rs/core/src/tools/handlers/multi_agents/watchdog_self_close.rs` (+19 / -7)
  - `codex-rs/tui/src/chatwidget/slash_dispatch.rs` (+54 / -1)

## Verdict

**needs-discussion**

## Specific refs

- `codex-rs/core/src/agent/control.rs:1703-1716` — `fork_previous_response_id_enabled()` now defaults to `true` when `CODEX_EXPERIMENTAL_FORK_PREVIOUS_RESPONSE_ID` is **unset** (`value.is_none_or(...)`). Previously an unset env var meant disabled. This silently flips a previously experimental opt-in to opt-out for every user who ever upgrades; that's the kind of behavior change that needs a release note rather than just a test (`control_tests.rs:212-215` — `fork_previous_response_id_is_enabled_by_default`).
- `codex-rs/core/src/session/session.rs:228-235` plus `:391-455` — new `CODEX_MATERIALIZE_EPHEMERAL_ROLLOUTS` env var bypasses the ephemeral fast-path so rollouts are written to disk for debugging. The truthy-parser accepts anything except `""`/`"0"`/`"false"`, which is more permissive than the parser five lines away in `control.rs`. Two truthy-parsers in the same crate that disagree on `"yes"` / `"on"` is a small but real footgun.
- `codex-rs/core/src/tools/handlers/multi_agents/watchdog_self_close.rs:67-77` — adds `finish_watchdog_helper` then `finish_watchdog_helper_thread` after the existing close. The first call is fire-and-forget (`.await` with no error handling); only the second is `?`-propagated. Worth confirming that's intentional and not a missed `map_err`.

## Rationale

The diff is a cleanly-split "restore upstreamable subset" PR and the test additions are appropriate. The blocking concern is the default-flip in `fork_previous_response_id_enabled`: changing an env-gated experiment to default-on is a user-visible behavior change for anyone with rollout-fork side effects, and the PR body explicitly says fork-only release defaults are intentionally **not** in this stack — yet this does flip the default. That's worth a maintainer ack before merge; either it's a real intentional graduation (which deserves a CHANGELOG entry) or it's a leak from the `frodex/129-stacked` branch that should stay disabled-by-default. The dual env-parser inconsistency is a minor cleanup that could be folded into the same revision.

