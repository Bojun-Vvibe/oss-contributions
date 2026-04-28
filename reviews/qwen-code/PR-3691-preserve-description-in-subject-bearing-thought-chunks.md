# QwenLM/qwen-code#3691 — fix(cli): preserve description in subject-bearing thought chunks

- **Repo:** [QwenLM/qwen-code](https://github.com/QwenLM/qwen-code)
- **PR:** [#3691](https://github.com/QwenLM/qwen-code/pull/3691)
- **Head SHA:** `ba4f57b638942651a8eaa9b6b1c3a6105fea037f`
- **Size:** +51 / -3 across 2 files
- **State:** MERGED

## Context

Silent-data-drop bug in the streaming-reasoning UI. When an
OpenAI-compatible reasoning provider emits batched SSE deltas of the form
`**Subject**\n\nFirstWord ...` (a single chunk carrying both the closing
`**` of the heading and the first body token), the Thought-event handler
in `useGeminiStream.ts` was routing on the *parsed subject* alone — calling
`setThought(...)` to update the loading-indicator title and discarding the
parsed `description`. Result: "FirstWord" never made it into the persistent
reasoning text, the rendered output read "user mentioned globally..."
instead of "The user mentioned globally...", and there was no error
anywhere in the loop. Per-prompt-flake (depends on how aggressively the
provider batched), invisible to logs.

## Design analysis

The author's diagnosis (in the PR body) is precise:

> The Thought event handler routed only to `setThought` and discarded the
> description. As a result, the first body word that happened to share a
> chunk with the closing `**` was silently dropped from the persistent
> reasoning display. The handler now treats subject-only chunks as
> discrete loading-indicator updates and routes everything carrying
> streamed text through the throttled buffer; the existing flush merger
> preserves the subject.

The fix narrative is "subject-only → loading indicator (synchronous);
description-bearing (with or without a subject) → throttled buffer; the
merger at `useGeminiStream.ts:1255-1260` already preserves the subject
when batched chunks land in the same flush window."

The merger pattern referenced is:

```ts
subject: queuedThought.value.subject || mergedThought.subject
```

That's the load-bearing piece that lets this fix work without duplicating
state — the subject from the first chunk survives the throttled merge with
the body of subsequent chunks.

The test (`useGeminiStream.test.tsx:2737-2780`) is well-shaped:

```ts
yield {
  type: ServerGeminiEventType.Thought,
  value: { subject: 'Evaluating installation approach', description: 'The' },
};
yield {
  type: ServerGeminiEventType.Thought,
  value: { subject: '', description: ' user mentioned globally installed qwen,' },
};
// …
await waitFor(() => {
  expect(mockAddItem).toHaveBeenCalledWith(
    expect.objectContaining({
      type: 'gemini_thought',
      text: 'The user mentioned globally installed qwen,',
    }),
    expect.any(Number),
  );
});
expect(result.current.thought).toEqual({
  subject: 'Evaluating installation approach',
  description: 'The user mentioned globally installed qwen,',
});
```

Three things this test gets right:

1. **Reproduces the exact two-chunk pattern from the bug report.** First
   chunk has both `subject` and `description: 'The'`; second chunk has
   `subject: ''` and the rest of the body. That's the actual SSE shape
   the user observed, not a synthetic worst-case.
2. **Asserts on both `mockAddItem` (the persistent history) AND
   `result.current.thought` (the live indicator state).** Catches the
   case where a future refactor preserves the description in the history
   but breaks the subject in the indicator (or vice versa).
3. **Uses leading-space concatenation** (`'The'` + `' user mentioned...'`
   → `'The user mentioned...'`) — this hits the *real* tokenization shape
   of OpenAI-compatible reasoning models (BPE tokens have leading spaces),
   not the squashed-no-space worst case. The PR body explicitly notes the
   no-leading-space edge (`Theuser`) is a separate `parseThought` issue
   that doesn't arise in practice.

## Risks / nits

1. **Throttle delay introduced for previously-synchronous descriptions.**
   The PR body acknowledges this as the main tradeoff: descriptions in
   subject+description chunks now wait up to `STREAM_UPDATE_THROTTLE_MS`
   before rendering, matching every other reasoning chunk. That's a
   consistency win — the previous synchronous render of subject+description
   chunks was the *outlier* — but if any UX test was timing-sensitive
   on first-render-after-subject, it would need updating.
2. **No test for the subject-only chunk path.** The fix preserves
   subject-only chunks as discrete synchronous updates, which is the
   loading-indicator responsiveness story. A second test asserting that a
   `{subject: 'Foo', description: ''}` chunk lands in `result.current.thought`
   immediately (not after throttle) would lock the discrimination boundary.
3. **`parseThought` trim-edge issue documented but explicitly out of
   scope.** The PR body calls out the
   `**Subject**\n\nThe ` + `user ` → `Theuser` squash that lives in
   `packages/core/src/utils/thoughtUtils.ts:50-53` but doesn't fix it,
   reasoning that real BPE tokenizers don't emit space-less leading
   tokens. That's a defensible scope-cut; worth a tracking issue so it
   doesn't get lost.

## Verdict

**merge-as-is.** (PR is already MERGED.) Tight diagnosis, minimal fix
that uses an existing merger primitive, regression test that mirrors the
exact bug shape, and an honest scope-cut on the related `parseThought`
edge. Exactly the way silent-data-drop bugs should be fixed.

## What I learned

- Silent data drops in streaming UI surface as flaky reasoning text
  ("sometimes the first word is missing"). Hard to reproduce, hard to
  log. The mitigation is per-provider-shape regression tests that pin
  the exact SSE chunk pattern observed in the wild.
- When two paths (synchronous indicator updates vs throttled body
  rendering) share a state structure, routing the discriminator on
  *what's present in the chunk* (`description` non-empty → throttled,
  else → synchronous) rather than *what's parseable* keeps the routing
  independent of the parser's internal state. Cleaner than the previous
  "parsed subject means subject-only" inference.
- BPE tokens almost always carry leading spaces; designing
  parser-edge-case tests around the "no leading space" pathological case
  often produces tests that don't reflect real provider output. Document
  the assumption explicitly.
