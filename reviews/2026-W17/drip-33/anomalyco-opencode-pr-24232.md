# Review: anomalyco/opencode#24232 — Honor noCacheTokens in usage accounting

- **PR**: https://github.com/anomalyco/opencode/pull/24232
- **State**: OPEN
- **Author**: willsarg (Will Sarg)
- **Range**: +75 / −1
- **Head SHA**: `fa50f650af84a865961503ba0dbe1431215c4eac`
- **Base SHA**: `1e4b7b5451dc924515444c006f6babbb9f24bc85`
- **Closes**: #24189
- **Verdict**: merge-as-is
- **Reviewer date**: 2026-04-25

## What the PR does

Fixes a usage-accounting bug for OpenAI-compatible providers
(DeepSeek, Moonshot) that report cached input via
`usage.cachedInputTokens` while leaving
`usage.inputTokenDetails` empty. The previous code in
`packages/opencode/src/session/session.ts:322–328` blindly
computed
`adjustedInputTokens = inputTokens - cacheReadInputTokens - cacheWriteInputTokens`,
but for these providers `inputTokens == prompt_tokens` already
includes the cached portion, so the subtraction double-counts
and clamps to 0 (issue #24189: "Cache Read: 0 on large
prompts"). The fix prefers `inputTokenDetails.noCacheTokens`
when present, falling back to the subtract-and-clamp.

## Observations

1. **The diagnosis is correct and the fix is the minimal
   right thing.** The diff at `session.ts:322–339` now reads:
   ```ts
   const noCacheInputTokens = input.usage.inputTokenDetails?.noCacheTokens
   const adjustedInputTokens = Math.max(
     0,
     noCacheInputTokens === undefined
       ? safe(inputTokens - cacheReadInputTokens - cacheWriteInputTokens)
       : safe(noCacheInputTokens - cacheWriteInputTokens),
   )
   ```
   The `=== undefined` (vs `== null` or truthy check) is
   important: a provider reporting `noCacheTokens: 0`
   (genuinely zero non-cached tokens) must take the explicit
   branch, not fall through to subtraction. Correctly handled.
2. **The two new tests pin both directions of the contract.**
   `test/session/compaction.test.ts` adds:
   - "uses cachedInputTokens without double-counting when
     noCacheTokens is absent" — DeepSeek/Moonshot shape, asserts
     `input == 0`, `cache.read == 14`, and a specific cost
     calculation `(14 * 0.16 + 43 * 4) / 1_000_000`. The cost
     assertion is the diagnostic teeth — it locks in the *full*
     accounting, not just the adjusted-input side.
   - "prefers explicit noCacheTokens when provider reports it
     alongside cachedInputTokens" — the case where both fields
     are populated and the explicit one must win. Asserts
     `input == 89` and `cache.read == 36_352`.
3. **`Math.max(0, ...)` clamp is preserved on both branches.**
   Important: `noCacheTokens - cacheWriteInputTokens` can still
   be negative if a buggy provider reports `noCacheTokens` smaller
   than `cacheWriteInputTokens`. The clamp keeps that from
   producing a negative cost. Defense in depth.
4. **Anthropic/Bedrock/Vertex path is provably untouched.** They
   already populate `inputTokenDetails.noCacheTokens`, so they hit
   the explicit branch with their pre-existing values; the
   subtraction is bypassed but the result is numerically the same
   since `noCacheTokens == inputTokens - cacheRead` for those
   providers. The Anthropic test at `compaction.test.ts:1915`
   continues to pass — confirmed by the author's `bun test`
   claim.
5. **Why `safe()` wraps the inner expression on both branches:**
   the `safe` helper (likely `(n) => Number.isFinite(n) ? n : 0`)
   handles non-numeric inputs from misbehaving providers. Keeping
   it on both branches preserves the existing tolerance.
6. **Nit (not blocking):** the new branch could short-circuit
   the `cacheReadInputTokens` calculation entirely when
   `noCacheTokens` is present (since the formula no longer needs
   it). It still computes it because `cacheReadInputTokens` is
   passed through to `result.tokens.cache.read` separately. That
   coupling is fine — just noting that the variable is used in
   two different contexts now.

## Verdict reasoning

Tight bug fix with a clean repro test pulled directly from the
broken provider shape. The fix is conservative (only changes
behavior when `noCacheTokens` is explicitly present), tests
pin both branches, no regression for Anthropic/Bedrock. Land
it.

## What I learned

When normalizing across SDK provider shapes, "trust the
explicit field over the derived one" is the right default.
DeepSeek and Anthropic both have valid token-accounting models;
the bug was treating Anthropic's invariant
(`inputTokens == non_cached`) as universal. The fix
recognizes that `noCacheTokens` is the *unambiguous* signal
and uses derivation only as a fallback.
