# google-gemini/gemini-cli #26352 — fix(core): filter unsupported multimodal types from tool responses

- **Repo:** google-gemini/gemini-cli
- **PR:** #26352
- **URL:** https://github.com/google-gemini/gemini-cli/pull/26352
- **Head SHA:** `77c7d7a7fedd4854ad15e071aeae965f622ae496`
- **Files touched:**
  - `packages/core/src/utils/generateContentResponseUtilities.ts` (+33 -5)
  - `packages/core/src/utils/generateContentResponseUtilities.test.ts` (+31 -0)
- **Verdict:** merge-after-nits

## Summary

Tools that return `audio/*` or `video/*` `inlineData` parts caused the
Gemini API to reject the entire turn (the API does not accept those
MIME types in `functionResponse` parts). This PR strips audio/video
inline parts before they're attached to the function response and
injects a "steering message" into `textParts` instructing the model to
reference the file via `@path` so the next turn picks it up as a
proper multimodal user-attached part.

## Specific notes

- **`generateContentResponseUtilities.ts:92-105`** — partition into
  `unsupportedMimeTypes` and `filteredInlineDataParts`. Only
  `audio/`/`video/` are stripped; `image/*` and other types still flow
  through. Correct narrow scope.
- **Lines 108-117** — the steering message is `unshift`ed onto
  `textParts`, so it precedes any model-generated text. Phrasing
  `[SYSTEM ERROR: PROTOCOL_LIMITATION]` plus `ACTION REQUIRED:` reads
  as instruction-override content. That's the intended pattern for
  Gemini agents but worth noting that this string is now part of the
  *visible* tool output the model sees on every audio/video tool
  return — if a tool legitimately returns an audio thumbnail alongside
  text, the model will see the steering preamble even when the rest of
  the response is fine.
- **Lines 131-135 (and 154-156)** — the rest of the function
  consistently switches to `filteredInlineDataParts.length`, so the
  "binary content provided (N items)" fallback message reflects the
  post-filter count. Consistent.
- **Test at `generateContentResponseUtilities.test.ts:161-188`** —
  asserts both that the binary parts are gone *and* that the steering
  string is present (with the specific MIME types listed). Good
  multi-aspect assertion.

## Nits

- The steering message is hardcoded English. For a CLI shipping in
  many locales this is fine for now (model-facing prompt), but consider
  marking it as such with an `// LLM-facing prompt; do not localize`
  comment.
- The `@path/to/file.mp3` example refers to a file path that may not
  exist in the user's workspace — the model could be confused. Maybe
  re-phrase as `@<path-to-the-file-on-disk>`.
- `unsupportedMimeTypes` could be hoisted to a module-level
  `const UNSUPPORTED_FUNCTION_RESPONSE_MIME_PREFIXES = ['audio/',
  'video/']` so future additions (e.g. `application/octet-stream`?) are
  one-liner edits.
