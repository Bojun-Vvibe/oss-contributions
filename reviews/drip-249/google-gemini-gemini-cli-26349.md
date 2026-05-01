# google-gemini/gemini-cli #26349 — fix(core): properly format markdown in AskUser tool by unescaping newlines

- URL: https://github.com/google-gemini/gemini-cli/pull/26349
- Head SHA: `9d8d7197f32a3f658ca0a7534d3435ab7f0a9d33`
- Files: `packages/core/src/tools/ask-user.ts` (+30/-1), `packages/core/src/tools/ask-user.test.ts` (+80/-1)
- Closes #25023

## Context / problem

When the model emits an `ask_user` tool call with multi-line markdown (headers, bullet lists, etc.), JSON encoding renders the newlines as literal `\n` *escape sequences* in the string field rather than actual `0x0A` bytes. The `MarkdownDisplay` renderer treats the string as one long single-line input — block-level markdown elements never trigger because there are no real newlines to delimit them. End users see a wall of text where they should see a formatted prompt. This is a generic LLM-tool-output failure mode (model double-escapes; framework doesn't unescape) — `ask_user` happens to be the one that surfaces it visibly.

## Design analysis

The fix at `ask-user.ts:96-126` lives in `createInvocation`, which is the right boundary — it intercepts the `params` after JSON-decode but before construction of `AskUserInvocation`, so all downstream consumers (the dialog, the markdown renderer, telemetry) see normalized strings without each having to know to unescape.

```ts
const unescape = (str: string): string =>
  str.replace(/\\r\\n/g, '\n').replace(/\\n/g, '\n');

const normalizedParams: AskUserParams = {
  questions: params.questions.map((q) => {
    const normalizedQ: Question = {
      ...q,
      type: q.type,
      question: unescape(q.question),
    };
    if (q.header) normalizedQ.header = unescape(q.header);
    if (q.placeholder) normalizedQ.placeholder = unescape(q.placeholder);
    if (q.options) {
      normalizedQ.options = q.options.map((opt) => ({
        ...opt,
        label: unescape(opt.label),
        description: opt.description?.trim() ? unescape(opt.description.trim()) : undefined,
      }));
    }
    return normalizedQ;
  }),
};
```

Strengths:
- **Order matters**: `\\r\\n` is replaced first, then `\\n`. If they ran in the other order, `\\n` would consume the `\n` half of `\\r\\n` and leave a stray `\r` behind. Correct ordering.
- **Field coverage** is exhaustive for the user-facing surface: `question` (always), `header`/`placeholder` (truthiness-gated, preserving the "absent → still absent" contract), `options[].label` (always when options present), `options[].description` (truthy-trim-gated to also collapse all-whitespace descriptions to `undefined` — possibly intentional cleanup, but worth flagging as a behavior change vs. pre-PR where an all-whitespace description would have been preserved as-is).
- **Spread preservation** (`...q`, `...opt`) keeps any unknown future fields on the question/option intact — won't silently drop a new field added to `Question` later.

### Tests at `ask-user.test.ts:71-145`
Two cases:
1. Full coverage of every unescaped field: `question`, `header`, `placeholder`, `options[0].label`, `options[0].description` all asserted to contain `\n` (real newline) after passing through `\\n` (escaped). Locks the field-coverage contract.
2. Mixed `\\r\\n` and literal `\n`: input `'Line 1\\r\\nLine 2\nLine 3'` should produce `'Line 1\nLine 2\nLine 3'`. This is the dispositive test for the ordering invariant — if the regex order were swapped, the test would fail with a stray `\r`.

The test reaches into `tool as unknown as { createInvocation: ... }` to get past the `protected` modifier, which is ugly but tolerable since `createInvocation` is the real boundary being tested.

## Risks

- **Behavior change on whitespace-only descriptions** at `:113`: `opt.description?.trim() ? unescape(opt.description.trim()) : undefined`. Pre-PR an all-whitespace description would survive as a string of spaces; post-PR it becomes `undefined`. Probably desired (renderers treat `undefined` and whitespace differently), but should be called out in the PR body and ideally tested.
- **No `\\t` handling**: tab escapes in markdown content (e.g. for code blocks or aligned tables) won't be unescaped. Not a regression — pre-PR they weren't either — but the PR title says "properly format markdown" and tabs are part of that surface. Worth a follow-up.
- **Unicode escapes (`\\u00XX`) not handled**: again, not a regression, but a determined model could still emit hex-encoded newlines that bypass this normalization. Out of scope for this fix.
- **Idempotence**: running `unescape` twice on already-real-newline content is a no-op (the regex matches only the literal `\n` two-character sequence, not the real `0x0A`). Good.
- **Risk of false positives on intentional `\\n`**: if a user's question literally contains the four characters `\`, `n`, e.g. when discussing escape sequences in a prompt, those characters get rewritten to a newline. Probably acceptable trade given the use case (prompts to humans, not code samples), but worth a `// known false-positive on literal '\n' substrings` comment.

## Suggestions

- Add a test for the all-whitespace-description case to lock the `undefined` outcome.
- Add a comment at `:99` explaining the regex order (`\\r\\n` first to avoid leaving stray `\r`).
- Add a follow-up issue for `\\t` and `\\u00XX` handling if those become real complaints.

## Verdict

`merge-after-nits` — narrowly scoped fix at the correct boundary (`createInvocation` is the single chokepoint between JSON-decoded params and downstream consumers), with the right field coverage and the right regex ordering, well-tested for the dispositive ordering invariant. Wants the all-whitespace-description test + an order-preserving comment, plus PR-body callout of the description-coercion behavior change.
