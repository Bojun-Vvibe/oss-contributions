# Review — openai/codex #20824: Drive TUI service-tier commands from model metadata

- **PR:** https://github.com/openai/codex/pull/20824
- **Head SHA:** `ac7eaf11cc3003a01a1ef0a70f98a80261d82105`
- **Author:** aibrahim-oai
- **Files:** 23 changed (+969 / −554)
  - `codex-rs/tui/src/bottom_pane/slash_commands.rs` (+308/−40)
  - `codex-rs/tui/src/chatwidget/slash_dispatch.rs` (+187/−170)
  - `codex-rs/tui/src/bottom_pane/command_popup.rs` (+145/−132)
  - `codex-rs/tui/src/bottom_pane/chat_composer.rs` (+97/−68)
  - + 19 other tui/protocol files

## Verdict: `needs-discussion`

## Rationale

This is a substantial refactor: it pulls service-tier identity out of the static enum `codex_protocol::config_types::ServiceTier` and into a model-metadata-driven shape that ships through the API and is treated as `String`-like across the TUI. The mechanical edits — switching `service_tier` from `Copy`-able enum to a `Clone` value (`exec/src/lib.rs:1058`, `app_server_session.rs:1337/1369/1401`, `app/thread_routing.rs:607`), replacing the hardcoded `Some(ServiceTier::Fast)` with `Some(SERVICE_TIER_PRIORITY.into())` in `app/tests.rs:3708,4328`, and renaming `additional_speed_tiers` → `service_tiers` in `app_server_session.rs:1048` — are all consistent and look correct.

The reason to discuss rather than merge-after-nits: this changes the persistence/wire shape of `service_tier` from a closed enum to an open string-derived value, and the diff visible to me only shows ~200 lines. The full +969/−554 footprint suggests the change touches `ConfigEditsBuilder::set_service_tier`, the `slash_commands.rs` UI surface (+308 lines including new help text and grouping logic), and `slash_dispatch.rs` (+187/−170 — essentially a rewrite). I want to see (a) what happens when an old config file on disk still contains the legacy enum string `"fast"` after upgrade — does it round-trip as a `ServiceTier` string, or does deserialization fail; (b) how `command_popup.rs` (+145/−132) presents the new dynamic tier list when the model metadata returns zero or one tier; (c) whether `additional_permission_profile` paths cleanly handle missing service tiers.

The smaller signals are encouraging: error messages in `app/event_dispatch.rs:1289-1296` are reworded from "Fast mode" to "Service tier" terminology consistently, and the persistence flow now emits `format!("Service tier set to {}", service_tier_display_name(...))` which derives display from the same `service_tier_display_name` helper added in `chatwidget/service_tiers.rs:1-49`. That's a clean centralisation. But the size, the protocol-shape change, and the slash-command-surface rewrite mean this needs an explicit upgrade-path discussion + a note on backward compatibility for persisted configs before merging.

## Banned-string check

Diff scanned; no banned tokens present.
