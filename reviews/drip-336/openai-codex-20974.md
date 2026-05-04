# openai/codex #20974 — 3- Add service tier id to config

- **Head SHA reviewed:** `fe8c6887fcd11f830ba42dc2499c363ae54fca92`
- **Size:** +217 / -72 across ~14 files (Rust + generated TS/JSON schemas)
- **Verdict:** merge-after-nits

## Summary

Third PR in a 3-part series (after #20969 model metadata and #20971
session-protocol service tier strings). Adds a top-level
`service_tier_id` to `Config` and `ProfileV2`, sitting alongside the
legacy `service_tier` field. Precedence rule: `service_tier_id` wins
when both are set in config; otherwise the request-level
`service_tier` is used.

Generated artifacts are kept in sync in the same commit, which is
good — they often drift in this series.

## What I checked

- `codex-rs/app-server-protocol/src/protocol/v2.rs:734` — new
  `service_tier_id: Option<String>` added beside the existing
  `service_tier`. Field is plain `Option<String>` matching the legacy
  field's shape; no narrowed enum. That's deliberate per the PR body
  ("service tiers are now string IDs"), but lets garbage values reach
  the request layer; validation needs to live downstream.
- `codex-rs/app-server-protocol/schema/typescript/v2/Config.ts:22` and
  `ProfileV2.ts:18` — generated `service_tier_id: string | null` is
  appended right after `service_tier`, preserving stable field order.
  Good for downstream TS consumers.
- `codex-rs/app-server-protocol/schema/json/codex_app_server_protocol.schemas.json`
  and the v2 / `ConfigReadResponse.json` mirrors all gain matching
  `service_tier_id` entries (4 spots, all `["string","null"]`). All
  generated, consistent.
- `codex-rs/config/src/config_toml.rs:+4` and
  `codex-rs/config/src/profile_toml.rs:+3` add the TOML field. Worth
  confirming `#[serde(default)]` is on these so existing configs that
  don't set the field continue to deserialize.
- `codex-rs/app-server/src/codex_message_processor.rs` swaps 5 lines
  for 1 — looks like a `.map_or`/`.or` consolidation reading
  `service_tier_id` first then falling back to `service_tier`. The
  precedence direction matches the PR body, but a unit test pinning
  the precedence (both set, only id set, only legacy set, neither
  set) would lock this in.
- `codex-rs/core/config.schema.json:+8` — schema additions for the
  CLI's TOML config. Consistent with v2 protocol additions.

## Nits

1. PR title prefix `3-` is slack-style; the project's commit style on
   `main` favors `[codex] …` or conventional prefixes — squash-merge
   message should be reworded for the changelog.
2. The body promises "config edit support for writing service_tier_id"
   — confirm the corresponding `config edit` JSON-RPC handler also
   accepts the new key (the diff snippet I reviewed only shows the
   reader side in `codex_message_processor.rs`; the writer paths are
   in the larger diff body).
3. Worth adding a note in the migration/changelog: when both
   `service_tier` and `service_tier_id` are present, the *id* wins —
   users who set both in different profiles for the same model could
   be surprised.

## Risk

Low. Additive field with a documented precedence rule, all generated
schemas kept in lockstep, legacy field retained for back-compat.

## Recommendation

Merge after (a) confirming `#[serde(default)]` on the TOML structs and
(b) adding a precedence-pinning unit test. The series #20969 →
#20971 → #20974 should land together to keep clients consistent.
