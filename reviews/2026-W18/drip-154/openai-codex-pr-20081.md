# openai/codex #20081 — feat(turn-metadata): record `plugin_ids_used` per turn

- PR: https://github.com/openai/codex/pull/20081
- Head SHA: `a75168d31c17c2a9ebe72bc82c4d7cf627c15676`
- Files: `codex-rs/core/src/session/turn.rs` (+5 / -0), `codex-rs/core/src/turn_metadata.rs` (+42 / -17), `codex-rs/core/src/turn_metadata_tests.rs` (+57 / -0)

## Citations

- `turn.rs:274-280` — at the post-plugin-resolution boundary inside `run_turn`, calls `turn_context.turn_metadata_state.set_plugin_ids_used(mentioned_plugin_metadata.iter().map(|plugin| plugin.plugin_id.as_key()))`. Placement is after `mentioned_plugin_metadata` is populated from `PluginCapabilitySummary::telemetry_metadata` (line 274) — i.e. only plugins actually visible to telemetry are recorded, not every plugin loaded in the session.
- `turn_metadata.rs:22-24` — adds `const PLUGIN_IDS_USED_KEY: &str = "plugin_ids_used"` and `const RESERVED_DYNAMIC_METADATA_KEYS: [&str; 2] = [TURN_STARTED_AT_UNIX_MS_KEY, PLUGIN_IDS_USED_KEY]`. The reserved-key array is the central registry of "keys the runtime owns and must not be overwritten by `responsesapi_client_metadata`."
- `turn_metadata.rs:79-99` — `merge_turn_metadata` signature changes from `turn_started_at_unix_ms: Option<i64>` to `additional_metadata: &BTreeMap<String, Value>`. Loops `for (key, value) in additional_metadata { metadata.insert(key.clone(), value.clone()); }` then guards client-supplied keys with `if RESERVED_DYNAMIC_METADATA_KEYS.contains(&key.as_str()) { continue; }`. Behavior preservation: `turn_started_at_unix_ms` was already in the reserved list (single-element check before; two-element `contains` now).
- `turn_metadata.rs:173-174` — `TurnMetadataState` field rename `turn_started_at_unix_ms: Arc<RwLock<Option<i64>>>` → `additional_metadata: Arc<RwLock<BTreeMap<String, Value>>>`. `BTreeMap` (not `HashMap`) is the right choice for downstream JSON serialization stability — keys come out alphabetized so log lines grep-stable across turns.
- `turn_metadata.rs:266-294` — old `set_turn_started_at_unix_ms` rewritten to insert into the map. New `set_plugin_ids_used<I: IntoIterator<Item = String>>` inserts after deduping into a `BTreeSet<_>` (so duplicate plugin invocations within a turn collapse to one entry) and **removes** the key entirely on empty input — i.e. a turn that resolves zero plugins emits a header *without* `plugin_ids_used`, not with `plugin_ids_used: []`. That's the right shape for downstream consumers using `key in payload` semantics.
- `turn_metadata_tests.rs:151-178` — new `turn_metadata_state_includes_plugin_ids_used` test passes three plugin IDs with one duplicate (`slack@openai-curated`, `github@openai-curated`, `github@openai-curated`) and asserts the output is `["github@openai-curated", "slack@openai-curated"]` — pinning both the dedup and the alphabetical sort from `BTreeSet`.
- `turn_metadata_tests.rs:207-` (visible cutoff) — companion negative test `turn_metadata_state_ignores_client_plugin_ids_used_when_unused` asserts the key is *absent* from the header (not present-with-empty-value) when `set_plugin_ids_used` was never called. Mirrors the existing `turn_metadata_state_ignores_client_turn_started_at_unix_ms_before_start` test.

## Verdict

`merge-as-is`

## Reasoning

This is a refactor disguised as a feature, in the good direction. The old code had a dedicated `turn_started_at_unix_ms: Arc<RwLock<Option<i64>>>` field on `TurnMetadataState` plus a single-element reserved-key check (`if key == TURN_STARTED_AT_UNIX_MS_KEY { continue; }`) — two places to update every time a new runtime-owned dynamic metadata key was added. The PR collapses both into a `BTreeMap<String, Value>` plus a `RESERVED_DYNAMIC_METADATA_KEYS` array, then adds `plugin_ids_used` as the second user of that pattern. The next dynamic key (e.g. `tools_used`, `model_overrides`, etc.) is now a one-line addition — append to the array, add a setter. That's the right structural payoff for refactoring.

The dedup-via-BTreeSet at `turn_metadata.rs:280` is load-bearing in a non-obvious way: in agentic flows it's normal for the same plugin to be referenced in multiple consecutive tool calls within one turn (e.g. `github.list_prs` then `github.get_pr` both resolve to plugin `github@openai-curated`). Without dedup, the JSON would carry `["github", "github", "github", "slack"]` and downstream cardinality counts (e.g. "what % of turns used plugin X") would skew. Dedup at the producer side keeps the consumer side simple, and the alphabetical sort from `BTreeSet` makes log-line diffing tractable.

The reserved-keys mechanism is the right safety boundary. `responsesapi_client_metadata` is supplied by the upstream Responses API client and could in principle contain arbitrary keys; if a future client started sending `plugin_ids_used` directly (say for client-side override of "what the client thought it was calling"), the runtime-owned value would be silently overwritten. The reserved-key check at line 99 keeps the runtime as the source of truth for telemetry it produces, while still letting client metadata fill in everything else. Worth noting the reserved list lives inline rather than in a `pub(crate) const` exposed for tests — `turn_metadata_tests.rs` doesn't re-import it, just relies on observable behavior, which is the right test discipline.

The empty-input branch at line 281 (`if plugin_ids.is_empty() { additional_metadata.remove(PLUGIN_IDS_USED_KEY); return; }`) handles a subtle edge case: a turn that initially resolved plugins, then had them all removed (e.g. plugin disabled mid-turn — currently impossible but the API allows it). Without the explicit remove, the header would carry `plugin_ids_used: []` from the prior call's empty-but-set state, which collides with the "absent means no plugins" convention the negative test pins.

One design question I'd have raised pre-merge: `as_key()` is called on each plugin's `plugin_id` to produce the string form, but the diff doesn't show that method's contract. If `plugin_id::as_key()` ever changes its serialization (e.g. drops the `@curated` suffix), every downstream telemetry consumer breaks silently. A `#[deprecated]`/contract test on `as_key()` would harden this; absent that, future-Bojun reviewing turn 9 of "why did our plugin metrics go to zero" will find the cause faster if the test asserts the *literal string* `"github@openai-curated"` (which it does at line 178 — good).

Net: clean refactor, sensible new feature, right tests, minimal risk surface. Ship.
