# openai/codex PR #20824 — Drive TUI service-tier commands from model metadata

- Head SHA: `da721fcbc9bc0c39c770bbf9029bb119c2c05bca`
- URL: https://github.com/openai/codex/pull/20824
- Size: +791 / -481, 19 files
- Verdict: **merge-after-nits**

## What changes

Replaces the hardcoded "Fast mode on/off" toggle in the TUI with a
metadata-driven service-tier selector. `App::PersistServiceTierSelection`
in `codex-rs/tui/src/app/event_dispatch.rs` (lines 1163–1206) now formats
the status line by looking up `service_tier_display_name(current_model, tier)`
from the model catalog rather than mapping `Some(Fast) -> "on"`. Tests in
`codex-rs/tui/src/app/tests.rs` are updated to use the new
`ServiceTier::priority()` constructor (lines 3665, 4283, 4289) instead of
the removed `ServiceTier::Fast` variant. `model_preset_from_api_model`
(`app_server_session.rs:1076`) reads `model.service_tiers` instead of
`additional_speed_tiers`. `chat_composer.rs` introduces a new
`SlashCommandAction` enum (and `ServiceTierCommand`) to carry richer
slash-command payloads than the bare `SlashCommand` enum.

## What looks good

- The shift from a bool-flavored "fast on/off" to N typed tiers is
  exactly the right direction — the old `Fast` mode was already an awkward
  sentinel for "priority". The new `ServiceTier::priority()` factory keeps
  call sites readable.
- Status messages now read "Service tier set to <DisplayName>" / "Service
  tier cleared" — this is materially clearer in the TUI than "Fast mode
  set to on".
- Error logging is updated consistently (`tracing::error!` message is
  changed from "fast mode" to "service tier" — see lines 1194, 1198, 1200).

## Nits / requests

1. `service_tier.clone()` is called twice in
   `event_dispatch.rs` (lines 1166 and 1168). `ServiceTier` is a small
   enum/wrapper — clones are cheap — but the second `.clone()` is
   unnecessary if the assignment to `self.config.service_tier` happens
   after the `set_service_tier` call. Trivial.
2. `service_tier_display_name(self.chat_widget.current_model(), service_tier)`
   borrows `current_model()` immediately followed by a re-borrow of
   `chat_widget` for `add_info_message`. Confirm `current_model()` returns
   an owned `String` or `&str` that doesn't conflict with the later
   mutable borrow — should be fine but worth an explicit bind.
3. The `SlashCommandAction` rename ripples through `InputResult::Command`
   (chat_composer.rs:264). Any external consumer pattern-matching on
   `InputResult::Command(SlashCommand)` will break — this is internal,
   but worth a `CHANGELOG` line.
4. Schema file `codex_app_server_protocol.schemas.json` keeps
   `additionalSpeedTiers` as `deprecated: true` — good. Confirm the
   server actually still emits it for one release for backward compat.

## Risk

Medium. This PR sits at the bottom of a stack (#20823, #20822, #20812)
and changes both protocol schema and TUI behavior. Recommend landing the
schema-only changes (#20823) first and confirming no downstream client
panics on `serviceTiers: []` defaults.
