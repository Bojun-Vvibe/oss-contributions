# PR #24411 — fix(opencode): avoid invalid Kilo reasoning details

- **Repo**: sst/opencode
- **PR**: #24411
- **Head SHA**: `101d34c42325bdc21ca2ad63efb7cfc891f2f460`
- **Author**: joshbochu
- **Size**: +129 / -2 across 2 files (33 src / 96 test)
- **Verdict**: **merge-as-is**

## Summary

When a model's `interleaved.field` is `"reasoning_details"` (used
by Kilo's Kimi-K2.6 routing among others), the previous code was
synthesizing a *string* `reasoning_details` value out of
`reasoningParts.map(p => p.text).join("")`. But Kilo's API
expects `reasoning_details` to be a **structured array** like
`[{ type: "reasoning.text", text: "..." }]`, not a plain string.
The mismatch produced 400s on any continuation turn that
included assistant reasoning. This PR splits the
`reasoning_details` path off from the `reasoning_content` path.

## Specific changes

- `packages/opencode/src/provider/transform.ts:201-231` — the
  `field === "reasoning_details"` branch now:
  1. Reads existing structured details from
     `msg.providerOptions?.openaiCompatible?.reasoning_details`,
  2. Concatenates with structured details extracted from each
     reasoning part's *own* `providerOptions.openaiCompatible
     .reasoning_details` (preserving type-tagged shape),
  3. If the merged array is empty, drops the field entirely
     instead of writing back an empty string,
  4. Otherwise writes the merged array back.
- `transform.ts:236` — comment correctly narrowed from
  "reasoning_content | reasoning_details" to just
  "reasoning_content" because that branch now only handles the
  string-shaped `reasoning_content` case. Good docstring
  hygiene.
- `transform.ts:204-207` — `Array.isArray(existingDetails) ?
  existingDetails : []` and the symmetric guard for per-part
  details defends against shape drift if upstream ever returns
  a non-array. Defensive but cheap.
- `test/provider/transform.test.ts:982-1080` — two new tests
  on a fabricated `kimiModel` (Kilo provider, `interleaved:
  { field: "reasoning_details" }`):
  - "does not synthesize string reasoning_details from
    reasoning text" — confirms a `reasoning` part with no
    structured `providerOptions.reasoning_details` produces no
    `reasoning_details` on the outbound message at all (vs.
    the old behavior of producing a stringified version).
  - "preserves structured reasoning_details from provider
    options" — confirms a `reasoning` part *with* structured
    details has them merged through.

## Risks

Low. The only behavior change is in the `reasoning_details`
branch, which was already broken (string instead of array).
Tests are explicit about the two regimes (no-details and
with-details). One nit:

1. `if (reasoningDetails.length === 0) { return { ...msg,
   content: filteredContent } }` drops `reasoning_details`
   from the providerOptions even if a *prior* assistant turn
   had set it. That's fine for the outbound shape (Kilo
   doesn't want an empty array either) but worth confirming
   it doesn't lose information that the next pass would have
   merged. Looking at the `existingDetails` read at L204 —
   the merge already includes those, so empty after merge =
   nothing to send = correct.

## Verdict

`merge-as-is` — focused fix for a real provider-shape mismatch,
with regression tests that pin both directions of the
behavior.

## What I learned

`reasoning_details` vs `reasoning_content` is the kind of
naming choice (singular "content" = string, plural "details"
= structured array) that *should* be a typed discriminator at
the type level, not two branches in a runtime conditional
that look almost the same. A future refactor could lift this
into a `ReasoningEnvelope` ADT with `Text(string)` and
`Structured(Array<ReasoningDetail>)` variants — and force the
provider config to declare which one it wants at compile
time.
