# anomalyco/opencode #24369 — feat(processor): add model fallback chain when retries are exhausted

- URL: https://github.com/anomalyco/opencode/pull/24369
- Head SHA: `87ea3667e4e61a4c7043171f203b59ae2347ef96`
- Verdict: **merge-after-nits**

## What it does

Adds an outer "switch to next model" loop around the existing
processor stream. When the primary model's retries are exhausted with a
*retryable* error (rate limits, 5xx, transient APIError), the processor
resolves the next model in `fallbackModels` via
`Provider.resolveFallbackChain` and re-enters the stream with that
model. AuthError, AbortedError, non-retryable APIError, and
ContextOverflowError explicitly do **not** trigger fallback — those
either need user intervention or already have a different remediation
(compaction).

## Diff notes

- `packages/opencode/src/provider/provider.ts:933-936` adds the new
  service method, and `:1683-1694` implements `resolveFallbackChain` as
  a sequential `Effect.exit` walk that returns the first resolvable
  `{ model, remaining }` pair. Skipping unresolvable entries (rather
  than failing the whole chain) is the right default — operators
  routinely list models that may not all be configured.
- `packages/opencode/src/session/message-v2.ts:391-401` pins the
  fallback chain on `User` schema as `Array<{providerID, modelID}>`.
  Schema-level rather than free-form is the right call.
- `packages/opencode/src/session/processor.ts:71-80` adds
  `shouldFallback` and `fallbackChain` onto `ProcessorContext`; the
  outer `while` loop in `process()` checks `ctx.shouldFallback`,
  resets error state, swaps the model, and re-enters.
- Test coverage in `test/session/fallback.test.ts` (222 lines) verifies
  the **negative** cases — auth 401 and non-retryable 400 both return
  "stop" without fallback. Good guardrails.

## Nits

- The PR description claims tests for the negative paths only; a
  positive-path test (primary 5xx → fallback to backup → success)
  would catch regressions in the outer-loop wiring more directly than
  the negative ones do. Worth adding before merge.
- `resolveFallbackChain` swallows the underlying `getModel` failure
  cause silently. Logging the reason at debug would help operators
  diagnose why an entry was skipped.

## Why this verdict

Design is sound — schema-typed chain, explicit error classification,
no mutation of the input chain (`slice(i + 1)` returns remaining).
Just want one happy-path test before this lands as a feature surface.
