---
pr: 3636
repo: QwenLM/qwen-code
sha: b1bfb2800638
verdict: merge-after-nits
date: 2026-04-26
---

# QwenLM/qwen-code #3636 — feat(core): cap concurrent in-flight requests per provider (#3409)

- **URL**: https://github.com/QwenLM/qwen-code/pull/3636
- **Author**: JahanzaibTayyab
- **Head SHA**: b1bfb2800638
- **Size**: +783/-13 across 9 files (`packages/core/src/{core/contentGenerator.ts,core/rateLimitedContentGenerator.ts,utils/concurrencyLimiter.ts}` + tests, `packages/cli/src/config/settingsSchema.ts`, `docs/users/configuration/settings.md`, `vscode-ide-companion/schemas/settings.schema.json`)

## Scope

Closes #3409 ("upstream returns `429 Too many concurrent requests for this model` under sub-agent fan-out"). Adds a FIFO concurrency limiter wrapping the content generator:

- **`ConcurrencyLimiter`** (`utils/concurrencyLimiter.ts`): standard token-pool primitive, exposed `.limit` getter for tests.
- **`RateLimitedContentGenerator`** (`core/rateLimitedContentGenerator.ts`): decorator over the base `ContentGenerator` that acquires before each call and releases on completion. Exposes `getLimiter()` for test assertions.
- **`createContentGenerator` wiring** (`core/contentGenerator.ts:266`): if `requestConcurrency > 0` either from `model.generationConfig.requestConcurrency` or env `QWEN_REQUEST_CONCURRENCY`, wrap with the limiter; otherwise pass through unchanged.
- **Settings surface**: `requestConcurrency` added to `settingsSchema.ts:914-924`, documented in `settings.md:46` ("Caps the number of in-flight requests… excess callers… wait FIFO instead. Set to `0` (default) for no limit"), and JSON-schema'd for the IDE companion.

Six new tests in `contentGenerator.test.ts:84-220` covering: no-wrap when unset, wrap when config set, env-var fallback when config unset, config wins over env, env parsing edge cases, and limiter.limit propagation through the wrapper.

## Specific findings

- **Right architectural shape: decorator over the generator** rather than a global semaphore in the dispatcher. This means concurrency is per-generator-instance, which lines up with the "per provider" intent — two configured providers get two independent limiters. Worth a JSDoc note on `RateLimitedContentGenerator` making this explicit, since "per provider" could otherwise be misread as "global to the process".

- **Env-var precedence is documented and tested**: config wins over env (`contentGenerator.test.ts:172-184`), env is fallback (`:154-167`), unset means no wrap (`:136-150`). The semantics ("0 / unset means unlimited") are correctly implemented and the test names are descriptive. Good.

- **FIFO claim needs verification.** The PR description and the docstring at `settings.md:46` both promise FIFO ordering. `ConcurrencyLimiter` would need to implement an explicit waiter queue (e.g. an array of `() => void` callbacks pushed on `.acquire()` when full, shifted on `.release()`) for FIFO. If the implementation is "promise-resolution-order", that depends on the JS event-loop microtask queue and is *usually* FIFO on V8 but not contractually so. Reviewer should `grep -n` the limiter implementation for an explicit queue; if it's a Promise-chain-based throttle, downgrade the FIFO claim in docs to "ordering is best-effort".

- **`process.env` mutation in tests at `contentGenerator.test.ts:110-118`** uses `afterEach` to restore `originalEnv`. Good — the previous-state save/restore pattern is correct. Spot-check that *every* test that mutates `QWEN_REQUEST_CONCURRENCY` is inside this `describe` block (so the `afterEach` fires); if any cross-block test sets the env, the leak across files is real.

- **`getLimiter()` and `getWrapped()` are visible test-only accessors.** `RateLimitedContentGenerator.getLimiter()` (`contentGenerator.test.ts:150,167,184`) and `LoggingContentGenerator.getWrapped()` (`:220`) appear to be production methods being used for white-box testing. That's fine but consider marking them `/** @internal */` or `/** @testonly */` so consumers don't build on them.

- **Error path: what happens to held tokens on rejection?** If the underlying generator throws, the wrapper must `release()` in a `finally` (or `try/finally`). The diff doesn't show the `RateLimitedContentGenerator` body — reviewer must confirm this, because a leaked token under repeated failures will silently lock the limiter at `inflight = limit` and the user sees "everything just hangs". This is the single highest-risk bug shape in concurrency wrappers.

- **Streaming generators**: `ContentGenerator` likely has both `generateContent` and `generateContentStream` (or equivalent). The limiter must wrap both, and for the stream case must release on stream-end *or* stream-error *or* upstream-cancel. If the limiter wraps only the synchronous "send request" call but not the long-lived stream lifetime, a single user with concurrency=4 and four hung streams will appear unlimited. Confirm in the implementation.

- **Cancellation**: if the caller aborts (AbortController), does the queued waiter get a chance to drop out, or does it stay in the queue holding the FIFO slot? Worth a test: enqueue beyond limit, abort one waiter, confirm later waiters still progress.

- **Settings naming**: `requestConcurrency` is fine, but consider whether `maxConcurrentRequests` would read better next to `maxRetries` and `timeout` in the same `generationConfig` block (shown at `settings.md:148-156`). Bikeshed — not a blocker, but the field is going to be in the user's settings.json forever.

- **Default `0 = unlimited` is a magic-value antipattern.** `null`/`undefined` for "unlimited" is more idiomatic; `0` for "unlimited" reads as "zero allowed" at every callsite that doesn't have the docstring in sight. The current code uses `requestConcurrency > 0` (`contentGenerator.ts:266`) so the impact is only ergonomic — but a user who sets `requestConcurrency: 0` thinking "block all requests" will be surprised. Consider rejecting `0` with a clear error and using `undefined` (or `null` or `-1`) for the unlimited sentinel.

- **`QWEN_REQUEST_CONCURRENCY` parsing**: env-var parsing tests at `:194` cover at least one edge case — confirm "non-numeric strings" and "negative integers" both fall through to "no limit" rather than throwing or coercing to NaN-then-Infinity.

- **Vscode IDE companion schema** is auto-regenerated to match the TS schema. Good — keeps the IDE auto-complete in sync.

## Risk

Medium-low for the gated path (only fires when explicitly configured), high for the unhappy paths (token leak on error, stream lifetime mismatch, abort handling). Most of the risk is buried in the implementation details of `ConcurrencyLimiter` and `RateLimitedContentGenerator`'s wrapping strategy, which the diff context shown doesn't fully expose.

## Nits

1. Confirm `try/finally` (or equivalent) around the wrapped call so `release()` always fires, even on error.
2. Confirm streaming generators wrap the *full* stream lifetime, not just the "send request" call.
3. Add an abort/cancellation test that confirms a waiter dropping out doesn't deadlock the FIFO queue.
4. JSDoc-clarify "per provider, per generator instance" on `RateLimitedContentGenerator`.
5. Verify FIFO is enforced by an explicit waiter queue, not Promise-resolution order; downgrade the docs claim if not.
6. Reject `requestConcurrency: 0` literal and use `undefined` for the unlimited sentinel (or document the magic value loudly).
7. Mark `getLimiter()` / `getWrapped()` as `@internal`.

## Verdict

**merge-after-nits** — the wiring, settings surface, env-fallback semantics, and test coverage of the configuration matrix are all good. Risk concentration is the limiter's error/stream/abort handling, which needs explicit review of the `RateLimitedContentGenerator` body before merge. FIFO claim and `0=unlimited` magic-value are doc/UX nits.

## What I learned

Concurrency limiters in TypeScript SDKs almost always ship with the same three latent bugs: (1) token leak on error path, (2) stream-lifetime vs request-lifetime confusion, (3) abort/cancellation deadlock. The first is caught by `try/finally`, the second by carefully picking which method to wrap (and which "completion event" releases), the third by a queue-aware abort handler. The configuration UX is usually correct; the bugs are below the wrapping boundary. When reviewing this kind of PR, the test names tell you the author thought about the config matrix; the absence of tests for "throws and re-acquires" / "stream cancelled mid-flight" / "abort while waiting" tell you what they didn't think about.
