# openai/codex PR #20823 — Expose structured service tiers in app-server

- URL: https://github.com/openai/codex/pull/20823
- Head SHA: `51368db8187bb6bf2807bd978e9a0ee793da2882`
- Author: aibrahim-oai (Ahmed Ibrahim)

## Summary

Adds a `serviceTiers: ModelServiceTier[]` field alongside the existing
`additionalSpeedTiers: string[]` on the `Model` schema in
`codex-rs/app-server-protocol/schema/json/codex_app_server_protocol.schemas.json`,
and marks the legacy field `deprecated: true`. New `ModelServiceTier`
struct carries `{ id: ServiceTierId, name, description }`. `ServiceTierId`
becomes an open string type. Same shape mirrored into the v2 schema and
the `v2/ModelListResponse.json`.

## Review

This is the additive half of a two-PR migration (paired with #20822 / #20824
which actually drive the TUI off the new field). Schema looks consistent:

- `additionalSpeedTiers` keeps its `default: []` and `string` items, so
  existing clients don't break (line ~11210 of the merged schema).
- `serviceTiers` lands with `default: []` so missing-on-the-wire is a
  non-event for old servers.
- `ModelServiceTier` requires all three fields (`id`, `name`,
  `description`) — confirm the backend always has a non-empty
  human-readable `description`; if not, making it optional would be safer
  than shipping `""`.
- `ServiceTierId` going from a closed enum (`fast | flex`) to an open
  string is the right call given the parallel #20812 change broadens it
  to `priority | flex | ultrafast | <future>`. The deprecation note on
  `additionalSpeedTiers` should explicitly say "removed in vN" so client
  authors know the runway.

Nit: the diff at the file tail shows `\ No newline at end of file` flipping
to a trailing newline. Good cleanup, just call it out so reviewers don't
trip over it.

## Verdict

`merge-as-is` — purely additive schema change with deprecation marker;
client-facing risk is bounded and the paired PRs cover the consumer side.
