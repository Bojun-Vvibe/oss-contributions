# openai/codex#21063 — add turn items view to app-server turns

- **Head SHA**: `82f46ee4fcff`
- **Verdict**: `merge-after-nits`

## Summary

Adds a new `TurnItemsView` enum (`notLoaded` | `summary` | `full`) and a required `itemsView` field on every `Turn` payload in the app-server protocol. Updates all three schema files (`ServerNotification.json`, `codex_app_server_protocol.schemas.json`, `codex_app_server_protocol.v2.schemas.json`) plus the v2 `ReviewStartResponse.json`. +613/-23.

## Findings

- `codex-rs/app-server-protocol/schema/json/ServerNotification.json:4231` and parallel sites in the other two schema files at `:17467` and `:15353` add `itemsView` to the `required` list of `Turn`. This is a **breaking schema change** — any existing client deserializing a `Turn` with strict required-field validation will reject payloads from older servers and vice versa. The PR description should call this out as a protocol bump (and ideally the wire schema version should be incremented if there is one).
- The new `TurnItemsView` definition (`:4307-4329` in ServerNotification.json) uses `oneOf` with three single-value `enum` strings. Functionally equivalent to a single `enum: ["notLoaded","summary","full"]`, but the `oneOf`-of-singletons shape is what `serde_json_schemars` typically generates from a Rust enum with per-variant docs. Acceptable; just note that some JSON-Schema clients (older Java, ajv-strict) handle this less gracefully than a flat `enum`.
- The replacement description on `items` (`:4213`: *"Thread items currently included in this turn payload."*) is an improvement over the old "Only populated on `thread/resume` or `thread/fork`…" comment, but readers now have to cross-reference `itemsView` to know when `items` is meaningful. Consider adding a one-liner to `items` like *"See `itemsView` for what subset is included."*
- v2 schema (`codex_app_server_protocol.v2.schemas.json:15470`) duplicates the `TurnItemsView` definition. That's expected for the v1/v2 split, but if there's a shared-types story planned, this is the moment to flag it.
- No diff-visible test coverage of the `notLoaded` / `summary` / `full` distinction in the schema PR itself — assuming Rust unit tests live in a sibling commit, otherwise add at least one round-trip test per variant.

## Recommendation

The shape is right and the new field name is clear. Before merge: (1) explicitly document the breaking-required-field change in the PR description, (2) cross-link `items` ↔ `itemsView` in the `items` description so generated docs are self-contained. The `oneOf` vs flat `enum` choice is consistent with the rest of the schema and not a blocker.
