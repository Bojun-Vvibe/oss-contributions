# anomalyco/opencode #24382 — feat(llm): auto-describe images via vision fallback

- URL: https://github.com/anomalyco/opencode/pull/24382
- Head SHA: `e18d6647de089ea8e0d0b78facbbfce62e34cd9b`
- State: OPEN
- Files: `config/config.ts`, `provider/provider.ts`, `session/llm.ts`,
  `test/fake/provider.ts` (+121/-5)
- Verdict: **merge-after-nits**

## What it does

When the active model can't read images (DeepSeek, GLM, Haiku, etc.)
but the user pasted a screenshot, the request previously failed with a
silent error. This PR routes image parts through a vision-capable
model first, replaces them with the text description, and then
forwards the message to the active model.

## Diff notes

Three pieces:

1. **Config knob** (`config/config.ts:142-148`) — new optional
   `vision_model: ConfigModelID` field with annotation. Auto-detected
   when unset.

2. **Provider lookup** (`provider/provider.ts:1647-1672`) — new
   `getVisionModel(current: Model)` Effect:
   - If `cfg.vision_model` is set, use it directly.
   - Otherwise scan all providers for any model with
     `capabilities.input.image === true`, **skipping the same
     provider as the current model**. The comment is honest about
     the reason: "if the user has run out of credits on that
     provider, picking another model from it would fail for the
     same reason."

3. **Fallback path** (`session/llm.ts:194-235`) — pre-`streamText`
   block detects `!input.model.capabilities.input.image &&
   hasImageParts(input.messages)`, runs `generateText` against the
   vision model with a fixed prompt
   ("Describe the content of the image(s) above..."), then
   `replaceImagePartsWithText` swaps every image part for a single
   `[Image description by ${visionModel.id}]:\n${description}` text
   block. The new `processedMessages` then flows into the existing
   isOpenaiOauth/isWorkflow branches.

## Risk surface / nits

- **Fail-soft on vision errors is correct but silent.** At
  `session/llm.ts:218-222`, an `Effect.tryPromise` catch logs the
  error and returns `null`, which causes the original `input.messages`
  (with image parts the active model can't handle) to be sent
  through. The user will then see *the original* failure they would
  have seen without this PR — but they'll have no signal that the
  fallback was attempted. Consider surfacing a tool-message-style
  notice into the conversation, or at least bubble the vision-side
  error to the operator UI.

- **Same-provider skip is too aggressive in non-credit-loss cases.**
  The "skip same provider" rule (`provider.ts:1664`) is correct for
  the credit-out scenario but wrong if a user is on, say,
  `openai/gpt-4o-mini` (no vision-capable in current setup) and has
  `openai/gpt-4o` installed. Worth gating behind a config like
  `vision_model_prefer_cross_provider: false` or at least making it
  fall back to same-provider when no cross-provider candidate
  exists. Right now the loop just returns `undefined`.

- **Multi-image semantics drop information.** In
  `replaceImagePartsWithText:101-118`, the *first* image part is
  replaced with the description and "subsequent image parts are
  dropped — already covered by the single description." This is
  fine when all images go to a single vision call (they do, see
  `extractImageParts` at `:80-89`), but the description is one blob
  that doesn't cite which image it's describing. For multi-image
  conversations the model loses positional grounding. Worth either
  asking the vision model to label "Image 1: ... Image 2: ...", or
  inserting a description text block per image position.

- **Prompt is hard-coded** ("Describe the content of the image(s)
  above in detail..."). Once any localization or domain-tuning
  arrives, this becomes a maintenance hot spot. Move to a constant
  with a TODO for `vision_fallback_prompt` config.

- **Test coverage**. The fake provider is updated to return
  `undefined` from `getVisionModel`, which exercises the "no fallback
  available" branch. There's no test of the *positive* path (vision
  model picks up the image, description is injected). Adding one with
  a recorded `generateText` mock would lock the contract.

## Why this verdict

Good idea, clean architectural placement (provider service exposes
the lookup; LLM service consumes it pre-stream). The merge nits are
silent failure surface, multi-image grounding loss, and missing
positive-path test. None are blocking; all are worth handling before
merge.
