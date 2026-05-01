# sst/opencode #25317 — fix: detect file MIME type from extension for -f attachments

- Link: https://github.com/sst/opencode/pull/25317
- Head SHA: `d754faa1e9f3f2af6c957a9b3b923006a3369e6e`
- Author: dominusbelial

## Summary
Closes the `-f`-attachment-becomes-text-garbage bug where every file, regardless of extension, was hardcoded as `text/plain` at `cli/cmd/run.ts:323`, sending PNGs/JPEGs/PDFs/MP4s to the model as their literal byte stream interpreted as UTF-8. The downstream pipeline at `prompt.ts:1155` already handled non-text MIME types correctly (binary read → base64 → `data:image/png;base64,...` URL → provider's `image_url` content block), so the fix is a one-line dispatch swap from the constant `"text/plain"` to a new `resolveMime(resolvedPath)` helper backed by a 25-entry static `MIME_MAP` covering the common image/video/audio/PDF extensions.

## Line-level observations
- `cli/cmd/run.ts:21-35` — `MIME_MAP` is a plain `Record<string, string>` lookup, extensions normalized via `path.extname(filePath).toLowerCase().slice(1)` so the dispatch is case-insensitive (`.PNG` and `.png` both resolve to `image/png`), with `?? "text/plain"` fallthrough so unknown extensions retain pre-PR behavior — zero risk to the existing text-attachment path.
- `cli/cmd/run.ts:338` — the directory-vs-file branch is preserved (`isDir(...) ? "application/x-directory" : resolveMime(...)`) so directory attachments aren't accidentally re-routed through the MIME map.
- The `resolveMime` definition is wedged between two `import` blocks at `:21-35` (after `WriteTool` import, before `WebSearchTool` import). ESLint with `import/first` would flag this — should be hoisted above all imports or moved to a `util/mime.ts` module so a future `bun lint` run doesn't bounce it.
- `MIME_MAP` covers PNG/JPEG/GIF/BMP/WEBP/ICO/TIFF/SVG/AVIF/APNG/JXL/HEIC/HEIF for images, MP4/WEBM/MOV/AVI for video, MP3/WAV/OGG/FLAC for audio, and PDF — but no `text/markdown` for `.md` (which `text/plain` swallows correctly), no `application/json` for `.json` (same), no `text/html` for `.html` (same). The narrow image-focused scope matches the PR's stated motivation (vision models) and the `?? "text/plain"` fallthrough keeps everything else working.
- No test file added — for a one-line dispatch swap on an extension lookup table this is defensible, but a one-`expect(resolveMime("/foo/bar.png")).toBe("image/png")` style test would lock the contract against future map regressions.
- The PR body's "Ollama vision via `image_url`" claim is structurally correct: the `data:image/png;base64,...` URL the existing pipeline produces IS the standard OpenAI `image_url` shape that Ollama's OpenAI-compatible endpoint handles.

## Verdict
`merge-after-nits` — fix the import-ordering layout (move `MIME_MAP`/`resolveMime` above all imports or extract to a util module) and add one unit test asserting `resolveMime` round-trips a representative entry per category. Behavior change is correct and minimal-surface.
