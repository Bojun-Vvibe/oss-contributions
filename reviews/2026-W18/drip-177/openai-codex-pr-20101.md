# openai/codex#20101 — install WFP filters for Windows sandbox setup

- **Repo:** openai/codex
- **PR:** [#20101](https://github.com/openai/codex/pull/20101)
- **Head SHA:** `3a320a8db59ad833b0913db994fc9e442c642c37`
- **Author:** iceweasel-oai
- **Size:** +768 / -1 across 8 files

## Summary

Adds a first wave of Windows Filtering Platform (WFP) filters to the Codex
Windows sandbox so the offline-account user can't reach arbitrary network
egress destinations. New `wfp.rs` module (~409 lines) wraps the WFP C API,
`wfp_setup.rs` integrates it into the existing setup orchestrator, and the
sandbox `SETUP_VERSION` bumps `5 → 6` so existing users re-run full setup
on next launch.

## What's actually going on

Three layers:

1. **WFP wrapper** (`codex-rs/windows-sandbox-rs/src/wfp.rs:1-409`). Opens a
   WFP engine session, registers a persistent provider+sublayer with stable
   Codex-owned GUIDs (`PROVIDER_KEY = 2e31d31c-...`, `SUBLAYER_KEY = e65054fd-...`),
   then iterates `FILTER_SPECS` and installs each filter scoped to the offline
   account's user SID via `FWPM_CONDITION_ALE_USER_ID`. All registrations
   carry `FWPM_PROVIDER_FLAG_PERSISTENT` / `FWPM_SUBLAYER_FLAG_PERSISTENT` /
   `FWPM_FILTER_FLAG_PERSISTENT` so they survive reboots.
2. **Setup integration** (`codex-rs/windows-sandbox-rs/src/setup_main_win.rs:609-617`).
   Inside the existing elevated full-setup path, after `ensure offline outbound
   block`, calls `install_wfp_filters(codex_home, offline_username,
   analytics_enabled, log_callback)`. Failure logs but doesn't abort setup —
   matches the PR body's "log failures non-fatally."
3. **Version bump** at `codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs:38`:
   `SETUP_VERSION: u32 = 5 → 6`. Existing users will see the version mismatch
   and re-run full setup, which picks up the new WFP install.

The `analytics_enabled` plumbing is the new contract field — added to `Payload`
with `#[serde(default = "default_analytics_enabled")]` (default `false`) at
`setup_main_win.rs:91-93,107-109`, and to `ElevationPayload` at
`setup_orchestrator.rs:421-424`. The orchestrator wires
`analytics_enabled: codex_otel::global().is_some()` at
`setup_orchestrator.rs:740` so the elevated child only emits OTel metrics if
the parent process actually has OTel configured.

## Specific line refs

- `codex-rs/windows-sandbox-rs/src/wfp.rs:71-87` — `install_wfp_filters_for_account`
  entry point: opens engine, begins WFP transaction, ensures provider/sublayer,
  iterates `FILTER_SPECS`, commits transaction. Transaction wrapping is the
  right discipline — a panic mid-install rolls back instead of leaving partial
  filter state.
- `codex-rs/windows-sandbox-rs/src/wfp.rs:74` — `delete_filter_if_present` before
  every `add_filter`. This is the upgrade-friendly pattern: re-running setup
  with modified `FILTER_SPECS` cleanly replaces filters by their stable GUID
  keys instead of accumulating duplicates.
- `codex-rs/windows-sandbox-rs/src/wfp.rs:62-65` — `PROVIDER_KEY` / `SUBLAYER_KEY`
  GUIDs marked "do not regenerate." Good comment; these are effectively part of
  the sandbox's persistent on-disk schema.
- `codex-rs/windows-sandbox-rs/src/setup_main_win.rs:609-617` — install call inside
  the `run_setup_full` branch, behind the `cfg(target_os = "windows")` gate that
  the surrounding block already provides.
- `codex-rs/windows-sandbox-rs/src/setup_main_win.rs:88-93` — `analytics_enabled`
  field with `#[serde(default = "default_analytics_enabled")]`.
- `codex-rs/windows-sandbox-rs/src/setup_main_win.rs:107-109` — `default_analytics_enabled()`
  returns `DEFAULT_ANALYTICS_ENABLED = false`.
- `codex-rs/windows-sandbox-rs/src/setup_main_win.rs:130-160` (new test mod) —
  two payload-roundtrip tests asserting the default and the explicit-true cases.
  Right surface for these tests; doesn't try to test WFP itself which would
  require an elevated test env.
- `codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs:38` — `SETUP_VERSION`
  bump to 6.
- `codex-rs/windows-sandbox-rs/src/setup_orchestrator.rs:740` — `analytics_enabled:
  codex_otel::global().is_some()` — the right gate, but see Reasoning below.
- `codex-rs/windows-sandbox-rs/Cargo.toml:33,77,80` — adds `codex-otel` workspace
  dep and `Win32_NetworkManagement_WindowsFilteringPlatform` /
  `Win32_System_Rpc` features to `windows-sys`.

## Reasoning

This is solid Windows-systems code. The transaction discipline, persistent-key
strategy, and "delete then add" upgrade pattern are all correct shapes for WFP.
The choice to scope filters by `FWPM_CONDITION_ALE_USER_ID` against the offline
account's SID rather than process-wide is the right blast-radius — admins or
the regular user account remain unaffected.

A few concerns worth flagging:

1. **Failure visibility.** The PR body says "log failures non-fatally" and the
   integration at `setup_main_win.rs:613-616` matches that — a `log_callback`
   gets called on each line. But there's no telemetry counter for "filter
   installation succeeded vs failed" beyond OTel-gated analytics. For a
   security-relevant feature, ops should be able to know "what fraction of
   sandbox setups successfully installed filter N" — without that signal,
   silent regional WFP regressions (a Windows update that changes filter
   semantics, etc.) won't be visible until users hit them. Consider an
   always-on counter in addition to the OTel gauge.

2. **`FWP_E_ALREADY_EXISTS` vs `FWP_E_FILTER_NOT_FOUND` semantics.** The diff
   imports both at `wfp.rs:18,20` but the snippet doesn't show the handling. If
   `delete_filter_if_present` swallows `FWP_E_FILTER_NOT_FOUND` (correct, "not
   present is fine") and `add_filter` swallows `FWP_E_ALREADY_EXISTS` (also
   correct in idempotent install), the design is right. Worth confirming the
   error-classification matrix in code review on the PR — these two error
   codes are easy to handle wrong and a swap silently leaks duplicate filters.

3. **`SETUP_VERSION` bump implies user-visible re-setup.** Bumping from 5 to 6
   means every existing Windows sandbox user will run full setup (with UAC
   elevation prompt) on next launch after upgrading. PR body acknowledges this.
   Consider a release-notes entry highlighting that — new prompt is a support
   surprise otherwise.

4. **Tests don't exercise WFP.** Understandable (would need elevated test
   harness), but the two payload tests at `setup_main_win.rs:130-160` only
   cover serde roundtripping. Some smoke-test infra in CI that runs the
   full elevated path against a Windows VM would catch regressions in the
   WFP-API call sequencing — out of scope for this PR but worth a follow-up
   tracking issue.

5. **`codex_otel::global().is_some()` at `setup_orchestrator.rs:740`.** This
   only captures whether OTel was configured at the moment the orchestrator
   built the payload — it doesn't reflect runtime changes (re-config during
   setup) or whether the elevated child process can actually re-establish the
   OTel exporter. That's probably fine for the analytics-flag semantics, but
   document it: "analytics_enabled is a snapshot of orchestrator-side OTel
   state at payload-build time."

## Verdict

**merge-after-nits** — confirm `delete_filter_if_present` swallows
`FWP_E_FILTER_NOT_FOUND` and `FWP_E_ALREADY_EXISTS` is handled correctly in
`add_filter` (the imports are there but the visible diff doesn't show the
matchers — a swap of the two would silently leak filter duplicates on
re-setup); add an always-on success/failure counter for WFP installation
distinct from the OTel-gated analytics gauge so ops can detect silent
WFP regressions on Windows updates without enabling full telemetry; add a
CHANGELOG / release-notes entry calling out the `SETUP_VERSION 5→6` bump
so the unexpected UAC prompt on next launch isn't a surprise; and file a
follow-up tracking issue for an elevated-Windows-VM CI smoke test that
exercises the full WFP install path end-to-end.
