# openai/codex #20939 — [codex] Render backend-selected near-limit prompts in TUI

- SHA: `9151fc21e2ee2e3ae681b2a5c4ec5927a84789e7`
- State: OPEN, +957/-5 across 12 files

## Summary

Wires backend-pushed `current_usage_limit_nudge` payloads into the TUI: live `AccountRateLimitsUpdated` notifications act as coarse 75%/90% prefetch triggers, the prefetched authoritative snapshot drives a one-shot proactive nudge per session, and a new feature flag (`current_usage_limit_nudge`) gates the whole surface (default-on, Stable). Ships ~600 lines of new tests covering staleness, prefetch dispatch, and "do not clobber authoritative state on coarse signal".

## Notes

- `codex-rs/features/src/lib.rs:226-227,1075-1080` — new `Feature::CurrentUsageLimitNudge` registered at `Stage::Stable, default_enabled: true`. Stable + default-on for a never-shipped feature is aggressive — Stable usually implies "we have field data". Recommend landing as `Stage::Beta, default_enabled: true` or `Stage::Stable, default_enabled: false` for one release window so the flag has a meaningful kill switch if a backend regression makes the nudge spammy.
- `codex-rs/tui/src/app/app_server_events.rs:78-79` — channel renamed from `on_rate_limit_snapshot(Some(...))` to `on_live_rate_limit_snapshot(...)`. The new name correctly distinguishes "live coarse signal from server" vs "authoritative refresh result", which is the central invariant the prefetch design relies on. Good rename.
- `codex-rs/tui/src/app/tests.rs:148-181` — `stale_rate_limit_refresh_results_are_ignored` covers the generation-counter staleness check. Drives gen=2 then gen=1, asserts the gen=2 nudge survives. This is the load-bearing correctness property; well-targeted test.
- `codex-rs/tui/src/app/tests.rs:183-260` — `account_rate_limits_updated_does_not_clear_authoritative_usage_nudge_state` is the right anti-regression test for the "coarse signal must not clobber authoritative state" contract. Without it, a single in-flight live notification arriving after a prefetch would erase the nudge.
- `codex-rs/tui/src/chatwidget/current_usage_limit_nudge.rs` (+137, new) — module is not visible in the abbreviated diff, but the test file `chatwidget/tests/current_usage_limit_nudge.rs` is +462 lines with 30+ scenarios per the file count. Solid coverage. Worth confirming the new module exposes a single `pub fn` surface vs leaking helpers.
- `codex-rs/tui/src/chatwidget.rs` adds +143/-1. The bulk of the gating logic lives here. From the imports added in `app/tests.rs:69-70` (`UsageLimitNudge`, `UsageLimitNudgeAction`), the protocol type is shared with `codex-app-server-protocol`. Worth a one-line note in the PR description that this PR depends on those types already being on the protocol crate (or a follow-up PR pinning the version).
- `codex-rs/tui/src/chatwidget/snapshots/codex_tui__chatwidget__tests__proactive_usage_prompt_variants.snap` (+35) — snapshot test locks the rendered prompt copy. Good. Reviewer should eyeball the snapshot in `gh pr diff` for tone/typos before approving.
- Magic numbers: 75%/90% thresholds appear in the PR description. If they're hardcoded in `current_usage_limit_nudge.rs`, surface them as named constants near the module top so the next backend tuning lands as a one-line change.
- No mention in the PR body of how this interacts with the existing `WorkspaceOwnerUsageNudge` feature (line 224, also a usage-nudge surface). Two simultaneous nudges-per-session sources risk double-display. Worth a sentence in the PR description about precedence.

## Verdict

`merge-after-nits` — strong test coverage, correct staleness handling, well-chosen rename. Re-tier the feature flag to Beta for one release, add a precedence note vs `WorkspaceOwnerUsageNudge`, and ensure 75/90 thresholds are named constants. The HEAD SHA has already moved once during review (PR list showed `18bcf76f...`, current is `9151fc21...`), so re-pin before final approve.
