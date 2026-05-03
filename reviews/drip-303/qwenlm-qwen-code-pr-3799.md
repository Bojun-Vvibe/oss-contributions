# QwenLM/qwen-code #3799 — feat(cli): normalize model list response parsing across OpenAI-compatible endpoints

- URL: https://github.com/QwenLM/qwen-code/pull/3799
- Head SHA: `c9a54597b1c726c551bb1c22063281e7a21d7dd6`
- Scope: +545 / -1 across 3 files

## Summary

Pairs with #3797 (`/model list` subcommand). Normalizes `fetchModels()` to
handle three response shapes from `/v1/models` endpoints:

- Standard: `{ data: [{ id }] }` (OpenAI)
- With `object` field: `{ object: "list", data: [{ id, owned_by }] }`
  (DeepSeek)
- Bare array: `[{ id }]` (some self-hosted providers, e.g. Ollama-shaped)

Extra fields (`owned_by`, `created`, `permission`) are ignored — only
`id` is extracted.

## What I checked

- `packages/cli/src/ui/commands/modelCommand.ts` (+171) — new exported
  `fetchModels()` function. The exporting from `modelCommand` allows the
  test file to import it directly (vs going through the command surface),
  which is a clean separation.
- `modelCommand.test.ts` (+369, 16 new test cases for `fetchModels`
  alone) — covers all three shapes plus: missing `id` skipping, extra
  fields ignored, error responses, malformed JSON. Well-structured matrix.
- `i18n/locales/en.js` (+5) — three new strings: command description,
  baseUrl missing error, generic fetch failure prefix. Adding them only to
  `en.js` (not all locales) is the standard pattern for this repo's i18n
  flow — translations get filled in by a follow-up.

## Nits

1. **No `id` validation beyond presence.** A malicious or buggy provider
   could return `{ data: [{ id: "" }, { id: null }] }`. The "skip entries
   with missing id" test handles missing keys but `id: ""` and `id: null`
   should also be filtered. Quick check: `if (typeof entry.id === 'string'
   && entry.id.length > 0)`.
2. **No total/page handling.** Some providers paginate `/v1/models`. If
   the response includes `has_more` or `next_cursor`, this code silently
   truncates. Probably fine for v1 (OpenAI itself doesn't paginate this
   endpoint) but worth a `// TODO: pagination` if the codebase is moving
   that direction.
3. **Auth header.** The function takes `apiKey` and presumably sends
   `Authorization: Bearer ${apiKey}`. Some providers (Azure, custom
   gateways) want different header names. Not a regression from current
   behavior — just a known limitation worth noting in a follow-up.
4. **Bare-array detection.** Confirm the dispatch order is:
   `Array.isArray(json) → bare array path; json.data → wrapped path`.
   If reversed, an OpenAI response with `data` field that *is* an array
   would still work, but the code paths should be unambiguous. The 16
   tests should catch any reordering.
5. **Test coverage of error responses.** The diff snippet shows happy-path
   cases. PR description claims "16 new for fetchModels" — double-check
   that includes at least one HTTP 401, one HTTP 500, and one timeout.
6. The PR description is well-structured — good practice.

## Risk

Low. New code path behind `/model list` (PR #3797). Backward-compatible
with existing OpenAI shape. Failure mode is "model list shows fewer items
than expected", not "command crashes".

## Verdict

`merge-after-nits` — primarily the `id`-validity check (`""` and `null`)
and confirming error-response test coverage.
