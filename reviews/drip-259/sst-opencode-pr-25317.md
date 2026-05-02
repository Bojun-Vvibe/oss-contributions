# sst/opencode PR #25317 — fix: detect file MIME type from extension for -f attachments

- PR: https://github.com/sst/opencode/pull/25317
- Head SHA: `d754faa1e9f3f2af6c957a9b3b923006a3369e6e`
- Author: @dominusbelial (Gustavo Poveda)
- Closes: #25353
- Size: +16 / -1

## Summary

`opencode run --file <path>` was hardcoding `mime = "text/plain"` for every non-directory attachment. The downstream pipeline in `session/prompt.ts` already knows how to base64-encode binary content and emit `data:image/png;base64,...` URLs for vision models — it just never got a non-text MIME type to trigger that path. So PNG / JPEG / PDF attachments via `-f` were silently turned into UTF-8-corrupted text blobs.

Fix is a 16-line `MIME_MAP` table plus a `resolveMime()` helper that does an extension lookup with a `text/plain` default.

## Specific references from the diff

- `packages/opencode/src/cli/cmd/run.ts:22-37` — the new `MIME_MAP` literal and `resolveMime()` helper.
- `packages/opencode/src/cli/cmd/run.ts:338` — the call site changes from `"text/plain"` to `resolveMime(resolvedPath)` (directories still resolve to `application/x-directory`).

## Verdict: `merge-after-nits`

Right fix at the right layer; no test, and the helper is placed mid-import-block which is structurally wrong.

## Nits / concerns

1. **Helper inserted between import statements.** Lines 22-37 (`MIME_MAP` + `resolveMime`) are sandwiched between `import { WriteTool } …` (line 21) and `import { WebSearchTool } …` (line 38). ESLint / `bun typecheck` may not flag this, but stylistically every other file in this package puts top-level consts and functions *below* the import block. Move the helper down to just above `RunCommand = cmd({ ... })`.
2. **`text/plain` default for unknown binaries is still wrong.** A `.bin` / `.dat` / `.parquet` attachment will now hit this code path, get `text/plain`, and the downstream pipeline will read it as text — same bug class, smaller blast radius. Consider `application/octet-stream` as the unknown default; the binary base64 path in `prompt.ts` should already handle that. (Worth at least a TODO if you don't want to expand scope.)
3. **No test added.** A one-shot unit test asserting `resolveMime("foo.PNG") === "image/png"` (uppercase) and `resolveMime("foo.unknownext") === "text/plain"` would lock both the case-folding and the default behavior. The case-folding path (`.toLowerCase()` at line 35) is easy to regress.
4. **Mime list is opinionated and partial.** No `webm` audio variant, no `mkv`, no `txt`/`md`/`json`/`csv` (which all *should* stay `text/plain` and do — fine, but worth documenting that the list is intentionally vision-model-focused).
