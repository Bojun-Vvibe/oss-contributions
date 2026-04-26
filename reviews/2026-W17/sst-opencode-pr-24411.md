# sst/opencode #24411 — fix(opencode): avoid invalid Kilo reasoning details

- **Repo**: sst/opencode
- **PR**: #24411
- **Author**: joshbochu (josh)
- **Head SHA**: 101d34c42325bdc21ca2ad63efb7cfc891f2f460
- **Base**: dev
- **Size**: +129 / −2 across `packages/opencode/src/provider/transform.ts`
  (+33/−2) and a new test block (+96/+0).

## What it changes

In `normalizeMessages` (`transform.ts:198-239`), splits the
"interleaved reasoning" branch by the `field` discriminator:

- **`reasoning_details` path** (Kilo / Moonshot Kimi K2.6): collect
  per-part `providerOptions.openaiCompatible.reasoning_details` arrays
  and concatenate with any existing message-level array. Set the
  resulting array on `message.providerOptions.openaiCompatible.reasoning_details`
  (or omit if empty).
- **`reasoning_content` path** (DeepSeek and others): unchanged —
  still concatenates `part.text` into a single string.

Previously both paths called `reasoningParts.map(part => part.text).join("")`
and stored the result. For `reasoning_details` providers, that string
was being placed where the upstream API expects an *array of
structured objects*, leading to validation rejection.

## Strengths

- Correct provider-aware branching driven by `capabilities.interleaved.field`.
  No new heuristics, just routing on the existing schema field.
- Preserves `existingDetails` from the message-level
  `providerOptions.openaiCompatible.reasoning_details` rather than
  clobbering them — important for SDK-emitted messages that already
  carry the array.
- `Array.isArray(details)` guard on the per-part details prevents
  malformed parts (e.g. `details: undefined` or `details: "string"`)
  from poisoning the concatenated array.
- 96-line test block exercises a realistic Kimi K2.6 model fixture
  (`moonshotai/kimi-k2.6`, `kilo` provider) with the full capability
  schema, asserting that string reasoning text is *not* synthesized
  into `reasoning_details`.

## Concerns / asks

- The old comment "Include reasoning_content | reasoning_details
  directly on the message" is updated to "Include reasoning_content
  directly" — good, but the new `reasoning_details` branch lacks an
  equivalent docstring explaining why the structure differs. A
  4-line comment block above the `if (field === "reasoning_details")`
  branch would help future maintainers.
- The branch returns a fully reconstructed `msg` object even when
  `reasoningDetails.length === 0`, but only the `content` is filtered.
  Consider keeping the existing message-level `reasoning_details`
  field if present (currently dropped because the provider-options
  block isn't re-attached). May or may not be intentional — worth
  asking the author.
- Test fixture uses `release_date: "2026-04-20"` and a
  `cost.cache.write` field. If the model schema changes, tests will
  break in noisy ways. Not blocking.
- No test for the "carry-over existing details + new part details"
  merge case (only the "string-not-synthesized" assertion is in the
  diff slice). That's the exact failure mode the existing-details
  spread guards against; an explicit test would lock it in.

## Verdict

`merge-after-nits` — well-targeted fix with good test coverage.
Asks: a docstring on the new branch, clarification on the
"empty details" message-shape question, and a merge-case test.
