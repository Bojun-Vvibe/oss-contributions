# google-gemini/gemini-cli #26349 — fix(core): properly format markdown in AskUser tool by unescaping newlines

- **Repo:** google-gemini/gemini-cli
- **PR:** #26349
- **Head SHA:** `9d8d7197f32a3f658ca0a7534d3435ab7f0a9d33`
- **Verdict:** request-changes

## Summary

Fixes #25023: when the LLM emits `\n` (literally backslash-n) inside
`question`/`header`/etc. fields of `ask_user`, the markdown renderer
sees two characters and renders them inline, breaking headers/lists.

Fix lives in `packages/core/src/tools/ask-user.ts:96-122`: a new
`createInvocation` override walks every `Question` and runs
`unescape(str)` on `question`, `header`, `placeholder`, `options[].label`,
`options[].description`. `unescape` is
`str.replace(/\\r\\n/g, '\n').replace(/\\n/g, '\n')`.

## Concerns (blocking)

- **`ask-user.ts:97`: lossy and over-eager.** The fix unescapes `\n`
  *unconditionally*, but legitimate markdown content can contain a
  literal backslash-n — a code block showing `printf("hi\n")`, a
  regex example, a Windows path snippet. Those will silently become
  newlines. The right fix is at the LLM-prompt / tool-schema layer
  (tell the model to send actual newlines) or to scope the unescape
  to whitespace-context positions. As written this trades one
  rendering bug for a content-corruption bug.
- **`ask-user.ts:97`: incomplete escape handling.** `\\n` (a literal
  backslash followed by n in the JSON, encoded as `\\\\n`) and `\\r`
  (without `\n`) aren't handled. If the LLM is over-escaping, it
  often over-escapes inconsistently across fields.
- **`ask-user.ts:117`: silent option-description elision.** The new
  branch reads
  `opt.description?.trim() ? unescape(opt.description.trim()) : undefined`.
  This *trims* the description as a side effect of the
  null-check — a description that was `"  important note  "` becomes
  `"important note"`. Innocent, but conflates "normalize escapes" with
  "trim whitespace" and isn't symmetric with how `header` /
  `placeholder` are treated.

## Nits

- **Tests (`ask-user.test.ts:71-138`):** good coverage of the
  happy path but no negative test for the code-block / regex /
  Windows-path case. Add one and you'll see why I'm uneasy.
- The cast `(tool as unknown as { createInvocation: ... })` (line
  82) is needed because `createInvocation` is `protected`. Consider
  exposing a `_createInvocationForTest` helper instead — the cast
  pattern erodes type safety long-term.

## Rationale

Real symptom, but the chosen fix corrupts legitimate user content
that contains escape sequences. Recommend scoping the unescape
(only inside detected markdown block contexts, or only when the
*entire* field is a single trailing/leading-whitespace-free escape
sequence) or moving the fix to the model-side prompt. Worth a
discussion before merging.
