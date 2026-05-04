# google-gemini/gemini-cli #26238 — Fix topic marker leakage in CLI output

- **Head SHA:** `7c0603ce0a76c343eddd0f061f9cfcf8422d23e1`
- **Size:** +6 / -4 across 4 files
- **Verdict:** **merge-after-nits**

## Summary
Two related fixes for "topic marker text leaks into the user-visible chat
stream":
1. Switches the active-topic marker emitted into the system prompt from the
   bracket form `[Active Topic: ...]` to a tagged form
   `<active_topic>\n...\n</active_topic>`
   (`packages/core/src/prompts/promptProvider.ts:283`), and tightens the
   sanitization regex from stripping `]` only to stripping `[]<>`
   (`promptProvider.ts:282`).
2. Adds an explicit instruction to the topic-update mandate in both the
   current and legacy prompts (`snippets.ts:659`, `snippets.legacy.ts:530`)
   telling the model **never** to emit topic markers in chat — they must go
   only through the `update_topic` tool.

## Strengths
- The bracket → angle-tag switch matches the rest of the prompt schema (the
  prompts file is full of `<...>` blocks already), so the marker now looks
  like other system instructions instead of like literal output the model
  might mimic. Good prompt hygiene.
- Sanitization on user-supplied `activeTopic` (`promptProvider.ts:281-282`)
  now strips both bracket families. This stops a topic title containing
  `<active_topic>` literally from breaking out of the marker block — small
  but a real injection vector for adversarial topic names.
- The added bullet at `snippets.ts:659` mentions both the legacy bracket form
  and the new tag form by name. That redundancy means the instruction stays
  correct even if the marker syntax changes again later.
- Test update in `promptProvider.test.ts:361, 372` is exact: it asserts the
  new tagged form is present (or absent) and would have caught the regression
  the PR is fixing.

## Nits
- The sanitization regex `/[\[\]<>]/g` (line 282) does not strip the slash,
  so a topic title `</active_topic>` would have its angle brackets removed
  but leave a stray `/active_topic` that visually looks like a closing tag.
  Edge case, but `/[\[\]<>\/]/g` would be safer; or, more robustly, base64-
  or JSON-encode the topic value before splicing it into the prompt.
- The sanitization runs on the raw `activeTopic` string but the
  `<active_topic>` block uses literal newlines around the value
  (line 283). If the model tokenizes `<active_topic>\n` as a unit, that is
  fine; if it ever sees the value on the same line as the tag, a topic title
  containing a `>` could still confuse it. Adding a short test for a topic
  title that contains `<`, `>`, and `]` would be one line and would lock
  this in.
- The duplication between `snippets.ts:659` and `snippets.legacy.ts:530` is
  unavoidable given the legacy/current split, but a comment cross-linking
  the two would help the next person who only updates one.

## Recommendation
Land after either widening the sanitization regex or adding a test for a
hostile topic title. The instruction-side fix is the bigger win and is
already correct as written.
