# PR #24857 — Fix: deepseek nvidia multimodal

- **Repo:** sst/opencode
- **Link:** https://github.com/sst/opencode/pull/24857
- **Author:** ezzy1630 (Ezzy Rappeport)
- **State:** OPEN
- **Head SHA:** `63f5cb86353451ac1f8da588c8e9e6c9d3dfc570`
- **Files:** `packages/opencode/src/provider/transform.ts` (+22), `packages/opencode/test/provider/transform.test.ts` (+77), `packages/opencode/test/tool/fixtures/models-api.json` (+36/-0), `bun.lock` (+2/-2)

## Context

When a user attaches an image to a message and the selected model is text-only, the existing `unsupportedParts()` rewrites each unsupported part into an `{ type: "text", text: "ERROR: Cannot read..." }` placeholder. That's correct behaviour — fail-loud, surface the limitation. But it leaves the message `content` as a 2+ element array of text parts. The downstream `@ai-sdk/openai-compatible` SDK serialises any array as a JSON array. NVIDIA's NIM Python backend pipes that array through `str.join()` and crashes with `sequence item N: expected str instance, list found`. So the user sees a cryptic 500 instead of a useful "image not supported" message.

## What changed

A new `mergeTextParts()` step in `transform.ts` runs after `unsupportedParts()`. For models whose capabilities don't include image/audio/video/pdf input, it concatenates an all-text `content` array into a single string scalar. Multimodal models are explicitly skipped. Two unit tests cover the happy path (text-only model, image attached → single string) and the negative (multimodal model unchanged). Two new model entries land in the fixture: DeepSeek V4 Flash and V4 Pro.

## Design analysis

The capability gate is the right hinge — you want the merge to be conditional on the model rather than the SDK or the provider, because there are multimodal models behind the same OpenAI-compatible adapter. Putting the merge in `transform.ts` after `unsupportedParts()` rather than inside it keeps the two concerns separable: "drop unsupported part types" and "normalise the wire shape for legacy backends" are different operations and may evolve independently. I'd suggest the function picks up a docstring explaining *why* (NVIDIA `str.join`) and not just *what*, because in 6 months someone will look at this and wonder if it's safe to remove.

One subtle concern in the test fixture changes: adding DeepSeek V4 Flash/Pro to `models-api.json` means any other test that snapshots that fixture will now drift. Worth verifying with `bun test` across the full suite that nothing else asserts the model count or full-fixture equality. The PR description says "all 141 existing transform tests pass" but doesn't mention the broader provider/tool suites.

## Risks

Two correctness edge cases worth checking:

1. What if the array contains a single text part (e.g., user sent text-only to a text-only model)? The merge should be a no-op and produce the same scalar shape the SDK would already emit — verify the early exit isn't accidentally creating a `{ text: "" }` for empty arrays.
2. What if `unsupportedParts()` produces a mix of text and a *supported* non-text part (e.g., text + audio for an audio-input model)? The capability check should keep that array intact; from the PR the gate is on missing image/audio/video/pdf inputs simultaneously, which seems right but should be tested.

The third commit (`63f5cb86`) casts the test fixture content to `any` to satisfy TS. That's a smell — usually means the public type for `content` is too narrow. If `Part[]` is the contract, the test shouldn't need a cast. Worth a 2-line look at the type to see if a discriminated union would let the test stay typed.

## Verdict

**Verdict:** merge-after-nits

Add a docstring on `mergeTextParts()` explaining the NVIDIA root cause; add the empty-array and mixed-parts edge tests; investigate whether the `as any` cast in the test is hiding a type narrowing miss.

---

*Reviewed by drip-155.*
