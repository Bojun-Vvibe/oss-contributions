# QwenLM/qwen-code #3636 — feat(core): cap concurrent in-flight requests per provider (#3409)

- **Head SHA:** `b1bfb28006383f3fa698da53d9831a5b73e05a0d`
- **Size:** +783 / -13 across 8 files
- **Verdict:** **merge-after-nits**

## Summary
Adds a `requestConcurrency` knob to per-provider `generationConfig` and a new
`ConcurrencyLimiter` + `RateLimitedContentGenerator` decorator that wraps the
underlying `ContentGenerator` to cap simultaneous in-flight requests. Callers
beyond the cap wait FIFO instead of getting upstream
`429 Too many concurrent requests for this model`. The limit can also be set
process-wide via `QWEN_REQUEST_CONCURRENCY`. Documented in
`docs/users/configuration/settings.md:177` and surfaced in the settings
schema (`packages/cli/src/config/settingsSchema.ts:917`). Wrapped only when
the value is a positive integer (`createContentGenerator` tests at
`packages/core/src/core/contentGenerator.test.ts:97`).

## Strengths
- The wrap-vs-no-wrap decision happens in `createContentGenerator` and the
  test suite covers all four cells (no env / no setting → not wrapped, env
  set → wrapped, setting set → wrapped, both set → setting wins). See
  `contentGenerator.test.ts:107-150` — exactly the right shape for a feature
  flag.
- Stream semantics are correct: the slot is released only after the stream is
  fully drained, not when `generateContentStream` returns
  (`rateLimitedContentGenerator.test.ts:228-260`). This is the subtle case
  most rate-limiter wrappers get wrong, and the test specifically asserts
  that a second stream call is parked until the first drains.
- The "releases the slot after rejection" test
  (`rateLimitedContentGenerator.test.ts:209-225`) catches the leaked-slot
  failure mode where an exception in the inner call would otherwise
  permanently consume capacity.
- Documentation is concrete about *why* a user would want this (the specific
  upstream 429 string, sub-agents firing in parallel, `/compress`
  interleaving) — `settings.md:177`. That kind of "here is the symptom you
  hit before you go searching for this knob" copy is rare and helpful.
- The env-var fallback (`QWEN_REQUEST_CONCURRENCY`) is documented as a
  process-wide override in `settingsSchema.ts:925`. Sensible escape hatch
  for ops who cannot edit `settings.json` per-deployment.

## Nits
- `peakInFlight = Math.max(this.peakInFlight, this.inFlight)`
  (`rateLimitedContentGenerator.test.ts:73`) measures the *inner* generator's
  peak, which is what we want, but the cap-respect assertion at line 196
  (`expect(inner.peakInFlight).toBeLessThanOrEqual(2)`) is racy in principle
  if the limiter's queue were ever non-FIFO. Fine in practice given the
  current semaphore implementation, but worth a comment.
- Default value in the schema is `undefined as number | undefined`
  (`settingsSchema.ts:921`). The doc says "0 / unset means unlimited", but
  the wrap predicate in `createContentGenerator` likely checks `> 0`. Worth
  confirming that explicitly setting `0` does not accidentally wrap with a
  zero-capacity limiter that deadlocks.
- The settings table column became significantly wider just from this one
  added phrase in `model.generationConfig` (the diff shows all the spacing
  recomputed). Consider moving the long prose into the dedicated
  `requestConcurrency` paragraph below the table and keeping the cell
  description short — current diff is mostly whitespace churn.
- No instrumentation: when a request is parked, there is no log line. For a
  user debugging "why is my agent slow", a single `verbose` log on
  acquire-with-wait would save a lot of guesswork. Optional.

## Recommendation
Land after confirming the `0`-vs-`undefined` semantics match the docs and
considering a low-volume "request parked at limit" log line. The core design
is right and the test coverage on the tricky stream-release path is
excellent.
