# openai/codex#20560 — feat: Track local paths for shared plugins

- **Author:** xl-openai
- **Head SHA:** `b08499511cd6a70f0bff39b13cd34840979dd973`
- **Base:** `main`
- **Size:** +589 / -76 across 14 files
- **Files changed:** `codex-rs/app-server-protocol/schema/json/codex_app_server_protocol.schemas.json`, `.../v2.schemas.json`, `.../v2/PluginShareListResponse.json`, `.../typescript/v2/PluginShareListItem.ts` (new), `.../typescript/v2/PluginShareListResponse.ts`, `.../typescript/v2/index.ts`, `codex-rs/app-server-protocol/src/protocol/v2.rs`, plus consumer plumbing

## Summary

Wire-protocol surface change: the `PluginShareList` response previously returned `Vec<PluginSummary>`, now returns `Vec<PluginShareListItem>` where each item bundles `{plugin: PluginSummary, share_url: String, local_plugin_path: Option<AbsolutePathBuf>}`. The new `local_plugin_path` field lets the desktop app correlate a remotely-shared plugin entry with a path on the local disk (the user's working copy of that plugin), enabling "open this shared plugin in your editor" UX without the client having to maintain its own side-table.

## Specific code references

- `protocol/v2.rs:4638`: response struct change `pub data: Vec<PluginSummary>` → `pub data: Vec<PluginShareListItem>`. This is the load-bearing wire-shape break — older clients that decode `data` as `Vec<PluginSummary>` will fail because the v2 server now returns objects with `plugin`/`shareUrl`/`localPluginPath` keys instead of the flat `PluginSummary` shape.
- `protocol/v2.rs:4653-4660`: new struct `PluginShareListItem { plugin: PluginSummary, share_url: String, local_plugin_path: Option<AbsolutePathBuf> }` with `#[serde(rename_all = "camelCase")]` so the TS export is `localPluginPath`/`shareUrl` consistent with the rest of v2. The optional `local_plugin_path` correctly serializes as `null` when the local copy isn't known (rather than missing-key), matching the `anyOf: [AbsolutePathBuf, null]` JSON-schema declaration at `codex_app_server_protocol.v2.schemas.json:8990-8997`.
- `schema/typescript/v2/PluginShareListItem.ts:1-7` (new): generated TS export `export type PluginShareListItem = { plugin: PluginSummary, shareUrl: string, localPluginPath: AbsolutePathBuf | null, };` — matches Rust shape exactly. Generation pipeline (ts-rs) is consistent with the rest of v2 schema/typescript/.
- `schema/typescript/v2/PluginShareListResponse.ts:4-6`: import swap `PluginSummary` → `PluginShareListItem` and `data: Array<PluginSummary>` → `data: Array<PluginShareListItem>`. Symmetric with the Rust change.
- `schema/typescript/v2/index.ts:286`: barrel re-export `export type { PluginShareListItem } from "./PluginShareListItem";` — alphabetically correct insertion between `PluginShareDeleteResponse` and `PluginShareListParams`, follows existing convention.
- `codex_app_server_protocol.schemas.json:12337-12361` and `codex_app_server_protocol.v2.schemas.json:8989-9013` and `v2/PluginShareListResponse.json:170-194`: triple-update of the JSON-schema dumps for the three top-level schema files. All three correctly carry the new `PluginShareListItem` definition with `required: ["plugin", "shareUrl"]` (omitting `localPluginPath` from required, consistent with the `Option<>` Rust shape). Three-way symmetric — no schema dump left stale.

## Reasoning

This is a wire-shape breaking change presented as a feature, which deserves explicit reviewer attention even though the v2 protocol may still be unstable. Three points to validate before merge:

1. **Backward compatibility story for the v2 wire shape.** Is v2 already considered "stable enough that we shouldn't break it" or is this still in the pre-stabilization window where shape additions like this are free? If the former, the response should add the field as a sibling (`Vec<PluginSummary>` + parallel metadata array) or use a versioned response variant rather than rewriting `data`'s element type. If the latter, the change is fine as-is. PR body should state this explicitly.

2. **`local_plugin_path` semantics across machines.** A plugin shared by user A and listed by user B will have `null` `localPluginPath` for B (B doesn't have it locally) and a path for A. This is the right behavior, but the field name reads as if it might be "the canonical local path on the sharing user's machine" — which would be a leak of A's filesystem layout to B. Doc comment at `protocol/v2.rs:4658` clarifying "local to the *requesting* client, computed server-side from local plugin index" would prevent the misread.

3. **42-file consumer fan-out** (only first 200 lines reviewed; the `gh pr view` reports 14 changed files but the diff trails off at 400 lines) — every consumer of the prior `data: Vec<PluginSummary>` response needs to be updated to destructure `item.plugin` instead of treating `item` as the summary directly. The TS index.ts shows clean propagation; verify the desktop consumer side has been updated symmetrically.

The schema-dump triple-update discipline (three JSON files all carrying the new definition) is the strongest signal this PR was done correctly — generation pipelines are easy to leave stale and this didn't.

## Verdict

**merge-after-nits**
