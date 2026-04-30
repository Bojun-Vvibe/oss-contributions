# QwenLM/qwen-code #3767 — fix(core): log the OpenAI request actually sent on the wire

- **Author:** tanzhenxin
- **SHA:** `d610e82`
- **State:** OPEN
- **Size:** +365 / -12 across `loggingContentGenerator/{loggingContentGenerator,loggingContentGenerator.test}.ts`, `openaiContentGenerator/{pipeline,pipeline.test,requestCaptureContext,...}.ts`
- **Verdict:** `merge-after-nits`

## Summary

Replaces the synthetic-reconstruction `--openai-logging` capture with the
actual on-the-wire request that the OpenAI SDK was called with. The pipeline
publishes the constructed request through a new `openaiRequestCaptureContext`
(an `AsyncLocalStorage`-backed setter) that the logger installs around its
`wrapped.generateContent[Stream]` call. If no capture happens (non-OpenAI
generators that bypass the pipeline), the logger falls back to the prior
`buildOpenAIRequestForLogging(req)` synthetic shape so headless flows still
get a non-empty log.

Plumbing concentrated in `loggingContentGenerator.ts:165-235`'s new
`startCaptureSession(isInternal)` returning a `{ wrap, resolve }` pair,
exercised across both `generateContent` and `generateContentStream` paths
including the error branches. Captured fields the synthetic path was silently
dropping: `extra_body` (and therefore `enable_thinking` / `thinking`),
DashScope `metadata`, `stream`, `stream_options`, and `samplingParams`
pass-through keys like `reasoning_effort`. Test coverage at
`loggingContentGenerator.test.ts:483-622` locks all three behaviors —
non-streaming captured-wins, streaming captured-wins, and synthetic fallback
when no capture happens.

## Reasoning

The bug being closed is a real one ("logs disagreed with the wire") and the
shape of the fix is correct — `AsyncLocalStorage` propagation across the
async boundary is the load-bearing assumption, the capture is set before
delegating to the wrapped generator and read after it resolves, and the
closure variable holds the captured request so the store itself does not
need to escape `.run()` (the PR body call out is right). The
`expect(loggedRequest).toBe(wireRequest)` identity assertion at
`:524,565` is the right strength — `toEqual` would let a structurally-
equal-but-different-object reconstruction sneak through. Three nits:

1. **The `startCaptureSession` API surface uses a `{ wrap, resolve }`
   tuple-returned-as-object instead of `using` / `Symbol.dispose`** —
   given the rest of the file is in the `try { wrap(...) } finally { ... }`
   shape anyway, an explicit-resource-management `Disposable` would prevent
   the `await session.resolve(req)` call from being forgotten in a future
   error branch (the existing code remembers it in all three places, but
   the contract is informal).

2. **Concurrent calls each get their own AsyncLocalStorage frame** is
   asserted by "verified by a dedicated test" in the PR body. Confirm
   the test is in the diff slice (it should live in
   `pipeline.test.ts`'s `openaiRequestCaptureContext integration` describe
   block at `:1782+`). Concurrent-isolation regression is the highest-value
   test for this class of `AsyncLocalStorage` change because it locks the
   property that's hardest to verify by inspection.

3. **The fallback path uses `buildOpenAIRequestForLogging(req)` only when
   `captured` is undefined**, but a partial-capture (e.g. the pipeline
   crashes mid-`convertGeminiRequestToOpenAI` and never calls the setter)
   silently produces an empty/incomplete log. A `captured ?? synthetic`
   assertion is fine for the happy path; for the error branch a logged
   field like `_capture_status: "captured" | "synthetic" | "partial"` would
   prevent the "log says X but X is not what was actually sent" failure
   class from regressing in a different way.

Overall a clean, targeted, well-tested fix to a real correctness gap. The
wire-format-fidelity property is the right one to lock, and the test surface
locks it correctly.
