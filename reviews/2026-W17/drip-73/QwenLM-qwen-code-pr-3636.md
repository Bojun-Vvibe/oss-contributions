---
pr: 3636
repo: QwenLM/qwen-code
sha: b1bfb280
verdict: merge-after-nits
date: 2026-04-26
---

# QwenLM/qwen-code #3636 — feat(generationConfig): cap concurrent in-flight requests per provider (#3409)

- **Author**: (community contributor)
- **Head SHA**: b1bfb280...
- **Size**: ~915 diff lines across `docs/users/configuration/settings.md`, `packages/cli/src/config/settingsSchema.ts`, `packages/core/src/core/contentGenerator.ts`, `packages/core/src/core/contentGenerator.test.ts`, and a new `packages/core/src/core/rateLimitedContentGenerator.ts`.

## Scope

Adds `model.generationConfig.requestConcurrency` (number, default unset = unlimited) plus a `QWEN_REQUEST_CONCURRENCY` env-var fallback. When set, `createContentGenerator` wraps the underlying `ContentGenerator` in a new `RateLimitedContentGenerator` that gates `generateContent`/`generateContentStream` calls through a FIFO semaphore so the upstream `429 Too many concurrent requests for this model` error never fires. Sub-agents firing in parallel and `/compress` interleaving with the main turn now wait FIFO instead of racing.

## Specific findings

- **Settings + docs + schema are aligned.** `docs/users/configuration/settings.md:153` adds the `requestConcurrency: 4` example to the JSON snippet, the explanatory paragraph after the table calls out the env-var fallback and "0 = unlimited" semantics, and `packages/cli/src/config/settingsSchema.ts:914-924` mirrors that as `parentKey: 'generationConfig'`, `showInDialog: false`, `requiresRestart: false`. Three surfaces agreeing is rare — good.
- **Wrapping policy is `LoggingContentGenerator(RateLimitedContentGenerator(real))`** — that layering is correct: the rate limiter sits *inside* the logging wrapper, so logged latencies include semaphore wait time (which is what an operator wants to see when diagnosing "why are my calls slow" — it surfaces the cap as the bottleneck instead of looking like network slowness).
- **Env-var parsing in `contentGenerator.test.ts:155-185`** — the table-driven test enumerates `'0'`, `'-1'`, `'nope'`, `'  '` all → unlimited. That's the right defensive parsing: any non-positive-integer is treated as "no cap," not a parse error. Add one more case for `'4.5'` (float) — does the limiter use 4 or 5 or treat as invalid? The test will document the choice.
- **`RateLimitedContentGenerator.getLimiter()` is exposed** for the test path that asserts `limiter.limit === 4`. That's fine for a test seam, but mark it `@internal` or move to a test-only export to discourage runtime callers from poking the limiter and resizing it.
- **Config-vs-env precedence** — the test at `:147-153` ("config value wins over the env var") is the right ordering. Make sure the precedence is documented in the doc paragraph as well; the current text says env var is a fallback "when the provider-level value is unset," which is right but easy to misread as "env always wins."
- **FIFO fairness** — the description claims FIFO ordering. Verify the underlying primitive (likely `p-limit` or a custom mutex queue) actually preserves arrival order. `p-limit` does; a naive `Promise.race` over a counter does not. If the implementation is custom, add a test that fires 10 concurrent calls with distinguishable resolve markers and asserts strict ordering.
- **Streaming cancellation** — what happens if a `generateContentStream` consumer drops the iterator before completion? Does the semaphore release? If the wrapper holds the slot until the upstream stream naturally ends, a buggy caller can deadlock the entire provider. Confirm there's a `try/finally` (or `using` disposable) around the slot acquisition that runs on early termination.
- **No-cap fast-path** — when `requestConcurrency` is 0/undefined, the test asserts the wrapper is *not* installed at all (no `RateLimitedContentGenerator` in the chain). That's the right optimization — zero overhead for the default case. Confirmed at `contentGenerator.test.ts:104-117`.
- **Docs nit** — `docs/users/configuration/settings.md:177` says "The environment variable `QWEN_REQUEST_CONCURRENCY` is honored as a fallback when the provider-level value is unset." Add: "Set to `0` (or any non-positive integer) to explicitly disable the cap even if the env var is set." Right now there's no documented way to *override* an inherited env var back to unlimited from `settings.json`.

## Risk

Low-medium. The change is well-tested, the wrapping order is right, the docs/schema/code are in sync, and the no-cap path is a true no-op. The two real risks are (1) streaming-cancellation slot leak and (2) FIFO claim being aspirational rather than tested — both addressable with one test each.

## Verdict

**merge-after-nits** — (1) add a streaming-cancellation test that asserts the semaphore slot is released when a stream consumer aborts; (2) add a FIFO-ordering assertion test; (3) add the float-parsing case (`'4.5'`) to the env-var table; (4) add the "set to 0 to override inherited env var" line to the docs; (5) mark `getLimiter()` `@internal`. The feature is well-scoped and the integration is clean.
