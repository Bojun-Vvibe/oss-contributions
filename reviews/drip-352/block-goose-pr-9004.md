# block/goose #9004 — support file-backed agent and skill editing

- URL: https://github.com/block/goose/pull/9004
- Head SHA: `fed3f4486e02a5d1afb157656d90d02ea8cece6f`
- Author: morgmart
- Size: +2841 / -1663 across 50+ files (Rust SDK + Tauri commands + goose2 React UI)

## Comments

1. `crates/goose-sdk/src/custom_requests.rs:744` — Doc comment changes from "portable JSON payload" to "canonical Markdown payload" but `ExportSourceResponse.json` field name is retained "for wire compatibility". This is the right call but **mark the field deprecated** with `#[deprecated(note = "field name retained for wire compat; contains markdown")]` so downstream consumers see the warning at compile time.
2. `crates/goose/src/sources.rs:180-185` (`export_source`) — Now returns the raw `SKILL.md` contents and a fixed `"SKILL.md"` filename. This is simpler, but the export will leak any local-only frontmatter fields (e.g. paths or tokens) that the previous JSON projection stripped. Please add a normalisation pass that whitelists frontmatter keys before export.
3. `sources.rs:188-205` (`import_sources`) — Removed the `version` envelope. Imports of legacy v1 JSON exports will now fail with `"Missing skill frontmatter"`. Add a fallback: if the payload starts with `{`, dispatch to a legacy-JSON parser and emit a deprecation log. Otherwise users who saved exports in the old format hit a wall.
4. `sources.rs:200-204` — `metadata.name.filter(|name| !name.trim().is_empty())` — good. But `description` is required only via `is_empty()` after `trim`; if frontmatter parses but lacks `description` entirely, the error path is via the unwrap on the parser. Please add an explicit `description.is_empty()` test alongside the existing `name` empty test.
5. `ui/goose2/src-tauri/src/commands/agents.rs:80-85` — Persona import now requires `.md` instead of `.json`. Same back-compat concern as (3): users with old `.json` persona exports get a hard error. Either accept both extensions or ship a one-shot migration.
6. The PR mixes three concerns: (a) Markdown-as-canonical export for skills, (b) persona import format change, (c) ~3k lines of UI churn around editor pages. Consider splitting (c) into a follow-up so the wire-format change can be reviewed and reverted independently if needed.
7. `ui/goose2/src/features/skills/api/skills.ts` and `useSkillImportExport.ts` — Diff window truncated; ensure the UI surfaces a helpful error message when import fails (not just a generic toast) so users know to convert their `.json` exports.

## Verdict

`request-changes`

## Reasoning

The direction (canonical Markdown export, file-backed editing) is correct and aligns with how skills are authored on disk. But this PR breaks read-compatibility with previously-exported `.skill.json` and `.json` persona files with no migration path, no deprecation window, and no fallback parser — users with saved exports will get a hard import failure on first try. Either keep the JSON reader for one release and emit a "deprecated, please re-export" warning, or ship a one-shot converter. Once the import path accepts both formats, this is mergeable.
