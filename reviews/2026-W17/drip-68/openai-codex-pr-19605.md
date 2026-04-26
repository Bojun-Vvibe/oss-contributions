# openai/codex #19605 — Delete unused ResponseItem::Message.end_turn

- **Author:** andmis
- **Head SHA:** `66d628637c6bf2fbc6ed96ad5be034999b3b887c`
- **Base:** main
- **Size:** +6 / -222 (51 files)
- **URL:** https://github.com/openai/codex/pull/19605

## Summary

Removes the `end_turn: Option<bool>` field from
`ResponseItem::Message` in `codex-rs/protocol/src/models.rs` and ripples
the deletion through every site that constructed or matched on the
struct. PR description is one sentence: "This field is unused. Delete
it." The field was there with a `#[serde(default,
skip_serializing_if = "Option::is_none")]` annotation and a comment
saying "Do not use directly, no available consistently across all
providers" — i.e. a vestigial provider-experiment field that never
became real.

## Specific findings

- `codex-rs/protocol/src/models.rs:690-696` — the canonical removal:
  the `end_turn: Option<bool>` field on `ResponseItem::Message` is
  deleted along with its `#[serde(default,
  skip_serializing_if = "Option::is_none")]`/`#[ts(optional)]` shroud
  and the explanatory comment ("Do not use directly, no available
  consistently across all providers"). Comment was load-bearing
  documentation that the field was a vestige; deletion makes the
  documentation moot.
- `codex-rs/protocol/src/models.rs:1043-1046` — matching cleanup in
  the `ResponseInputItem → ResponseItem` `From` impl drops `end_turn:
  None,` from the constructed `Message` variant.
- 5 schema files updated coherently:
  `codex-rs/app-server-protocol/schema/json/ClientRequest.json:2293-2298`,
  `codex_app_server_protocol.schemas.json:12740-12745`,
  `codex_app_server_protocol.v2.schemas.json:9414-9419`,
  `v2/RawResponseItemCompletedNotification.json:346-351`,
  `v2/ThreadResumeParams.json:757-762` — each removes the `"end_turn":
  {"type": ["boolean", "null"]}` block. (All five schema files also
  flip from "no trailing newline" to "trailing newline," which is
  unrelated noise but harmless.)
- `codex-rs/app-server-protocol/schema/typescript/ResponseItem.ts:14`
  — the TS type alias loses the `end_turn?: boolean` member from the
  `"message"` discriminant of `ResponseItem`.
- All other 43 changed files are mechanical removals of `end_turn:
  None,` from struct-literal initializations in tests and call sites
  (e.g. `compact_tests.rs:66`, `compact_tests.rs:75`,
  `thread_history.rs:3099`, `compaction.rs:137`,
  `thread_inject_items.rs:62`, `thread_inject_items.rs:198`,
  `thread_resume.rs:1658`, `thread_resume.rs:2605`,
  `codex_thread.rs:281`). Each removal is a single line in a struct
  literal with `phase: None,` left intact directly below — the diff
  shape is uniform and grep-auditable.
- The `protocol/src/items.rs` and `protocol/src/protocol.rs` and
  `state/src/extract.rs` and `tui/src/app/side.rs` touches are the
  same pattern in non-test code — wherever `ResponseItem::Message {
  ... }` was being built or destructured, the field is dropped.
- **Wire compat check (good):** because the field was `#[serde(default,
  skip_serializing_if = "Option::is_none")]`, omitting it on serialize
  is byte-equivalent to the previous always-`None` behavior. Older
  rollouts and resume payloads that *include* a literal `"end_turn":
  null` will still deserialize (serde silently ignores unknown fields by
  default in this codebase's pattern, but worth confirming the struct
  doesn't have `#[serde(deny_unknown_fields)]` — it doesn't, per
  inspection of the surrounding context).
- **Schema clients (small risk):** generated TypeScript / JSON-schema
  consumers that statically referenced `end_turn?: boolean` will simply
  see the property go away. None of the diffs show any code that *read*
  `end_turn` to drive logic, so client breakage is purely cosmetic.

## Verdict

`merge-as-is`

## Reasoning

This is exactly the kind of dead-field deletion that should ship: the
field was annotated as "do not use" since introduction, no code path
read it, and the deletion is one canonical struct change with the rest
being mechanical `end_turn: None,` removals in test fixtures and
constructors. Wire compatibility is preserved by the original
`skip_serializing_if = "Option::is_none"` annotation, so existing
serialized rollouts/resume payloads are still valid input. Schema files
are regenerated coherently across all five locations. The only
follow-up worth doing post-merge is a quick grep across the
`codex-rs/exec`, `codex-rs/codex-api/tests`, and any external
`codex-` SDKs for stale `end_turn` references — not a blocker for this
PR which is internally self-consistent.
