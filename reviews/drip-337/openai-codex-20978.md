# openai/codex #20978 — 4- Use model service tier slash commands

- **Head SHA reviewed:** `d718127935b791981777f0f92e536424314669c6`
- **Size:** +387 / -239 across 10+ files
- **Verdict:** merge-after-nits

## Summary

Removes the hardcoded `/fast` TUI slash command and replaces it with a
dynamic family of `/`-prefixed commands sourced from the selected
model's `service_tiers` metadata. Each tier shows up in the slash popup
with its description; selecting a tier persists the id, selecting it
again clears the override.

This is the 4th and last PR in a stack (after #20969, #20971, #20974).

## What I checked

- `codex-rs/tui/src/app/event_dispatch.rs:1252-1295` — the
  `PersistServiceTierSelection` event now carries an optional
  `service_tier_id` alongside the legacy `service_tier` enum, and
  resolves `config.service_tier_id` as `service_tier_id ?? service_tier
  .map(...request_value())`. The fall-back is correct, but the *name*
  collision (the field, the enum, and the local both share words) makes
  this hard to read. A comment above the assignment explaining "id wins
  if explicitly provided" would save future readers.
- `event_dispatch.rs:1284-1296` — the success-message branch flips from
  the binary "Fast mode set to on/off" to either "Service tier set to
  {id}" or "Service tier cleared". This is a user-visible string change
  for anyone who scripts against output; not a regression, but worth a
  one-line note in the PR body.
- `codex-rs/tui/src/bottom_pane/chat_composer.rs:267` adds an
  `InputResult::ServiceTierCommand` variant. Make sure all
  `match InputResult` sites in callers are exhaustive — Rust will catch
  it but a quick `rg "InputResult::"` to confirm everyone updated is
  worthwhile (the diff touches `chatwidget/slash_dispatch.rs` so most
  likely covered).
- `chat_composer.rs:404` introduces `service_tier_commands:
  Vec<ServiceTierCommand>`. The setter calls `self.sync_popups()` —
  good, the popup will redraw after the model swap. But the field is a
  plain `Vec`, not de-duplicated; if `set_service_tier_commands` is
  called twice with the same payload (e.g., model re-selection), you'll
  pay the redraw cost twice. Cheap enough today; flag as future opt.
- `chat_composer.rs:1734-1751` — the popup-completion path now matches
  on `CommandItem::Builtin(_) | CommandItem::ServiceTier(_)` and formats
  `selected_command_text` accordingly. The `Skills` early-return still
  short-circuits with `InputResult::Command(SlashCommand::Skills)`,
  which is correct. Good.
- The `fast_command_enabled` bool is removed cleanly from both struct
  and snapshot (`chat_composer.rs:392, 481, 572, 684`). No orphan
  references in the diff. Confirm grep against the rest of the crate
  to be sure no upstream still calls `set_fast_command_enabled`.

## Risk

Low-medium. The behavior change is intentional and matches the stack's
direction. Risk surface is mostly:

- Persisted config compat: existing users with `service_tier = Fast` in
  config still resolve correctly via the `or_else` fallback. Good.
- Telemetry / scripts that grep for "Fast mode set to on" will need
  updating.

## Recommendation

Merge after:

1. Add a comment to the `service_tier_id ?? service_tier.map(...)`
   resolution explaining precedence.
2. Mention the user-facing string change in the PR body so downstream
   integrations have a heads-up.
3. Run `cargo clippy --all-targets` on the touched crates — the
   removal of `fast_command_enabled` could leave dead `pub fn` getters
   elsewhere in the workspace.
