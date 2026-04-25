# PR: Delete unused ResponseItem::Message.end_turn

- URL: https://github.com/openai/codex/pull/19605
- Author: andmis
- Head SHA: `66d628637c6bf2fbc6ed96ad5be034999b3b887c`
- State: OPEN  (+6 / −222, 51 files)

## Summary

The `end_turn` field on the `Message` variant of `ResponseItem` was carried in the protocol but never consistently populated across providers (the comment at the old `protocol/src/models.rs:692-693` explicitly said "Do not use directly, no available consistently across all providers."). This PR deletes the field from the Rust struct, the TypeScript type export, the JSON Schemas (Client/Notification/v2/ThreadResumeParams), and removes the now-redundant `end_turn: None` initializers from ~45 call sites and tests.

## Specific observations

- `codex-rs/protocol/src/models.rs:690-693` — the `end_turn: Option<bool>` field and its self-deprecating comment are removed. This is a public-protocol struct: the change is breaking for any downstream consumer that destructured `ResponseItem::Message { id, role, content, end_turn, .. }` without a wildcard, and is observable to JSON consumers because the JSON schemas at `app-server-protocol/schema/json/*.json` no longer expose the field. The PR does not bump a protocol version and the PR body does not enumerate the breakage. For an OSS project shipping a `*.schemas.json` file, that is a SemVer-relevant change.
- `codex-rs/app-server-protocol/schema/typescript/ResponseItem.ts:14` — the exported TS type drops `end_turn?: boolean` from the message shape. Consumers writing against the published TypeScript bindings will get a compile error if they referenced the field, even though it was always `?`. That is the right outcome (the field was lying), but it should be called out in the PR description and the next changelog/release notes.
- `codex-rs/protocol/src/items.rs:269` and `codex-rs/protocol/src/protocol.rs:2870` (`From<CompactedItem> for ResponseItem`) and `codex-rs/app-server-protocol/src/protocol/thread_history.rs:3099` — the `end_turn: None` initializers are removed at every `ResponseItem::Message { … }` literal. That is a mechanical, complete sweep across the workspace; the lack of any remaining initializer in the diff is a good signal the cleanup is exhaustive.
- The 45-ish test-file changes (e.g. `codex-rs/core/tests/suite/{client,compact,review,…}.rs`) are pure noise removal of `end_turn: None` from struct literals. Low-risk.
- Schema deltas: each of `app-server-protocol/schema/json/{ClientRequest.json,…v2.schemas.json,v2/RawResponseItemCompletedNotification.json,v2/ThreadResumeParams.json}` has a 6-line removal at the same offset (the message-variant `end_turn` property and its `boolean` schema). Consistent. A consumer pinning an older schema and validating new payloads against it will not get the field — that's fine. A consumer pinning a *newer* schema and validating older replayed rollouts that contain `end_turn: true` will fail validation. The PR should mention whether rollouts in `~/.codex/sessions/` are forward-compatible.
- The PR is a pure deletion: no new tests are added (none needed for a field removal), and no migration is added for stored rollouts that may still contain `end_turn: true|false` keys. If `serde` deserialization is the strict variety, replaying an older rollout could now error with "unknown field end_turn". Worth grepping for `#[serde(deny_unknown_fields)]` on the message variant before merging — if it is present, this PR would silently break rollout replay.

## Verdict

`merge-after-nits`

## Reasoning

Removing a never-properly-populated field is the right call and the cleanup is mechanically thorough. The blocking question before merge is rollout-compatibility: stored sessions on user disks may contain `end_turn` keys, and depending on the serde attributes on `ResponseItem::Message`, deserialization could now reject them. Confirm `#[serde(deny_unknown_fields)]` is not in play (or add `#[serde(default, skip_deserializing)]` shim with a deprecation comment) and document the schema break in the PR body for downstream consumers of the published JSON/TS schemas.
