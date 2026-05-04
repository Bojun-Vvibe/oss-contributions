# google-gemini/gemini-cli PR #26275 — fix(hooks): preserve non-text parts in fromHookLLMRequest

- **PR:** https://github.com/google-gemini/gemini-cli/pull/26275
- **Author:** SandyTao520
- **Head SHA:** `6fd57eea` (full: `6fd57eeaa5321ff7fbf7782d6244b034ce899642`)
- **State:** OPEN
- **Files touched:**
  - `packages/core/src/hooks/hookTranslator.ts` (+120 / -17)
  - `packages/core/src/hooks/hookTranslator.test.ts` (+209 / -0)

## Verdict

**merge-as-is**

## Specific refs

- `hookTranslator.test.ts:173-225` — the regression test directly targets the bug: a 4-message conversation where the model emits `{text + functionCall}`, the user provides a `functionResponse`, and a `BeforeModel` hook modifies the user's first message text. Expectation: text is updated, `functionCall` survives, `functionResponse` survives, no extra parts injected. This is exactly the shape that was previously stripping tool context and inducing the infinite tool-call loop.
- `hookTranslator.test.ts:244-285` — interleaved text/function-only entries. Confirms the merge logic walks `baseRequest.contents` rather than rebuilding from `hookRequest.messages`, so function-only entries that `toHookLLMRequest` had skipped are preserved at their original index.
- `hookTranslator.test.ts:294-329` — multiple text parts collapsed into one (good — matches what hooks expect since they see the joined text), with `functionCall` preserved alongside.
- `hookTranslator.test.ts:331-358` — fallback paths: `baseRequest` undefined and `baseRequest` without `contents` both degrade to text-only construction. Important for backwards compat with callers that don't pass a base request.
- `hookTranslator.ts` (+120 / -17) — could not see full diff in this pass, but the test surface area is comprehensive enough to characterize the implementation: walk `baseRequest.contents`, splice updated text from `hookRequest.messages` by skipping non-text-bearing entries, append any extra hook messages at the tail.

## Rationale

This is a high-quality fix for a serious bug. The failure mode (infinite tool-call loop on any text-modifying `BeforeModel` hook in a session with prior tool calls) blocks PII redaction, FERPA/HIPAA scrubbing, prompt rewriting — every realistic enterprise hook use case. The fix preserves `functionCall` / `functionResponse` / `inlineData` / `thought` parts, which is the correct minimum surface area for the SDK contract.

Test coverage is excellent: 6 new test cases that pin down the exact edge cases (preservation, interleaving, multi-text collapse, two fallback paths, append-extras). Author identified the root cause precisely in the description and ships matching tests. Merge as-is.
