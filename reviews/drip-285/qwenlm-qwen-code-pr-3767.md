---
repo: QwenLM/qwen-code
pr: 3767
head_sha: d610e8216a6f9f95c0ef2592931bae8be11de39f
title: "fix(core): log the OpenAI request actually sent on the wire"
verdict: merge-after-nits
reviewed_at: 2026-05-03
---

# Review: QwenLM/qwen-code#3767 — `log the OpenAI request actually sent on the wire`

**Head SHA:** `d610e8216a6f9f95c0ef2592931bae8be11de39f`
**Stat:** +365 / −12 across 5 files in `packages/core/src/core/...`.

## Problem

`--openai-logging` previously rebuilt the request payload from the public
content-generator inputs and logged that synthetic version. That stripped
silently:

- `extra_body` (and therefore `enable_thinking` / `thinking` knobs)
- DashScope `metadata`
- `stream` / `stream_options`
- any other provider-injected sampling params (e.g. `reasoning_effort`)

So the log file never matched what actually went out on the wire — making
debugging "thinking-on vs thinking-off" or DashScope routing issues
basically impossible from logs alone.

## Approach

A new `requestCaptureContext.ts` exposes an `openaiRequestCaptureContext`
(an AsyncLocalStorage-backed slot) that stores a callback `(req) => void`.

In `pipeline.ts`, just before the actual OpenAI SDK call, the pipeline
runs the call inside `openaiRequestCaptureContext.run(setter, async () => ...)`,
and the SDK wrapper invokes `setter(actualRequest)` with the literal
request object passed to `client.chat.completions.create(...)`.

The logging decorator (`loggingContentGenerator.ts`) installs a setter
before delegating to the wrapped generator, then logs whatever the setter
captured. If nothing was captured (non-OpenAI path), it falls back to the
old reconstruction.

The new tests in `loggingContentGenerator.test.ts` verify the captured
payload for both `generateContent` (lines 481–525) and
`generateContentStream` (lines 527+) and assert object identity:

```ts
expect(loggedRequest).toBe(wireRequest);
expect(loggedRequest).toMatchObject({
  reasoning_effort: 'max',
  extra_body: { thinking: { type: 'enabled' }, enable_thinking: true },
  metadata: { dashscope_user_id: 'abc' },
});
```

The `toBe` (reference equality) is the important assertion — it locks in
"the logged object IS the wire object", not "the logged object equals a
shape that happens to look like it".

## Assessment

- AsyncLocalStorage is the right primitive here: it scopes the capture
  to the current async context, so concurrent requests can't cross-write
  each other's slots.
- Object-identity assertion in the test is the strongest possible
  guarantee against regression — if someone later "tidies up" by
  reintroducing a synthetic reconstruction, the test fails immediately.
- Falls back to old behavior when no capture is registered, so non-OpenAI
  generators are unaffected.

## Nits

1. **PII risk.** The whole point of this PR is that the captured request
   includes provider-injected fields like `metadata.dashscope_user_id`.
   That's a real user identifier going to the on-disk log file. The
   pre-existing logger probably already redacts the auth header, but
   it's worth either (a) running the captured request through the same
   redaction pass before writing, or (b) at minimum, documenting in
   `--openai-logging` help that the log now contains
   provider-attribution metadata.

2. **Memory holding.** The captured `wireRequest` is the literal object
   the SDK was called with — for streaming requests, that object stays
   referenced via the AsyncLocalStorage frame until the stream
   completes. For very long streams this is fine (the request is
   small) but it's worth confirming the capture context is dropped
   after the stream resolves so the request object can GC.

3. **Test fixture realism.** The streaming test uses
   `model: 'glm-5.1'` while `generateContent` uses
   `'deepseek-v4-pro'`. Adding one fixture that uses an actual Qwen
   model name with `extra_body.thinking` would lock down the most
   common production case for this project.

## Verdict

**`merge-after-nits`** — high-value debuggability fix done at the right
layer with strong tests. Nit #1 (PII redaction in the logger) is the
one I'd actually want addressed pre-merge; #2/#3 are polish.
