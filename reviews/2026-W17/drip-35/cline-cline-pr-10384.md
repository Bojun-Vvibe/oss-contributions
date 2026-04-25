# cline/cline #10384 — fix: cap retry-after delay to prevent silent multi-hour hangs

- **Repo**: cline/cline
- **PR**: [#10384](https://github.com/cline/cline/pull/10384)
- **Head SHA**: `65b1c5e2993f0cc7b8e3d1cc9b06300d57120b3e`
- **Author**: NgoQuocViet2001
- **State**: OPEN (+68 / -7)
- **Verdict**: `merge-as-is`

## Context

Issue #10139: when an LLM provider returns a 429 with a
`retry-after` header in the multi-hour range (Gemini quota exhaustion
typically returns 10800 seconds = 3 hours), the `withRetry` decorator
in `src/core/api/retry.ts` does an honest `setTimeout(delay)` for the
full duration. The user sees nothing — no toast, no error — for hours.
This is the worst-case UX for a rate-limit surface because there's no
way for the user to know whether to wait, switch providers, or abort.

## Design

The fix at `src/core/api/retry.ts:7-15`:

```typescript
const DEFAULT_OPTIONS: Required<RetryOptions> = {
    maxRetries: 3,
    baseDelay: 1_000,
    maxDelay: 10_000,
    maxRetryAfter: 60_000,   // ← new
    retryAllErrors: false,
}
```

A new `maxRetryAfter` option (default 60s) caps how long the
decorator will sleep on a `retry-after` hint. When the parsed delay
exceeds the cap, the decorator throws the original error immediately,
giving the upstream caller (the chat loop) the opportunity to
surface the message to the user, suggest a provider switch, etc.

The implementation logic is the right shape: the cap applies *only*
to delays computed from the `retry-after` header, not to the
exponential-backoff path. `baseDelay` / `maxDelay` continue to govern
non-header retries unchanged. So no behavior change on the common case
(transient 429 with no `retry-after` or with a small `retry-after`).

`60_000` (60s) as the default is a defensible Schelling point:
- Short enough that a user staring at a hung chat will see the
  failure within "I'd accept that" time
- Long enough to absorb genuine short-term rate-limit cooldowns
  without cascading into a chat-killing error

If maintainers want to tune, the option is per-call so individual
provider clients can opt into a higher cap (e.g., a long-batch job
might pass `maxRetryAfter: 600_000`).

## Test coverage

Two new test cases at `src/core/api/retry.test.ts:181-234`:

1. **`should throw immediately when retry-after exceeds maxRetryAfter`**
   — sets `retry-after: 10800` (3h), `maxRetryAfter: 60_000`, asserts
   the error throws on the first call (no retry) and `callCount === 1`.
   Exact regression for #10139.

2. **`should still retry when retry-after is within maxRetryAfter`** —
   sets `retry-after: 0.01` (10ms), asserts the decorator does retry
   and gets to success. Locks the negative case so a future
   "always throw on retry-after" regression would be caught.

The existing test at lines 112-145 was updated to use a stubbed
`Date.now()` and an explicit small delay (1s, well under default cap),
because its previous synthetic timestamp from `2010-01-01` produced
a delay that would now exceed the default cap and break the test.
That's a correct cascade-fix:

```typescript
const fakeNow = 1700000000000
sinon.stub(Date, "now").returns(fakeNow)
const retryTimestamp = fakeNow / 1000 + 1 // 1 second in the future
const baseDelay = 10000
```

The stub-via-`sinon.stub(Date, "now")` is the standard pattern for
this. Author also chose to swap `baseDelay` from 1000 to 10000 to keep
the original "header takes precedence over baseDelay" assertion
meaningful — good reasoning.

## Risks / Nits

1. **No test for an existing call site passing `maxRetryAfter:
   Infinity`.** The PR description mentions "Updated existing Unix
   timestamp test to pass `maxRetryAfter: Infinity` since its
   synthetic timestamp produces a delay that exceeds the default cap"
   — but the visible diff doesn't show that override; it shows the
   timestamp itself being made smaller. Either the description is
   slightly stale or the override is in a part of the diff I can't
   see. Either way, also having a test that explicitly verifies
   `maxRetryAfter: Infinity` disables the cap (and a 1-day delay is
   honored) would be a nice belt-and-suspenders. Optional.

2. **No call-site updates.** The default kicks in for every existing
   `@withRetry()` invocation across the codebase, so any provider
   client that legitimately wants to wait >60s on a `retry-after`
   needs to opt in. That's the right direction (fail loud by default,
   opt in to long sleeps), but worth a CHANGELOG line so providers
   know they may now surface 429s sooner.

3. **`retry-after` header parsing path is upstream of the cap** —
   the PR doesn't change parsing, just the decision after parsing.
   The Unix-timestamp variant (timestamp in the future) and the
   seconds-delta variant both flow into the same cap check. Good.

4. **`status = 429` field type annotation simplified** from `status:
   number = 429` to `status = 429` (line 19). TypeScript inference
   gets the same `number` type. Drive-by cleanup, harmless.

5. Nit: the cap is in milliseconds, but the option name
   `maxRetryAfter` doesn't disambiguate vs the `retry-after` header
   value (which is in seconds). A comment or a rename to
   `maxRetryAfterMs` would prevent a future contributor from passing
   `60` thinking they're capping at a minute.

## Verdict rationale

`merge-as-is`. Real bug fix, conservative default, surgical scope, two
new tests covering both sides of the cap, existing test cascade-fix
applied correctly. The naming nit (`maxRetryAfter` vs. `maxRetryAfterMs`)
is worth bikeshedding in a comment but not worth holding the fix.

## What I learned

`retry-after` headers are an under-trusted input. Servers can return
arbitrarily large values (hours, days), and naive `setTimeout` on
them is a denial-of-service against your own UI. The right pattern
is "cap by default, opt in to longer sleeps per call site," which is
exactly what this PR does. The cascade-fix on the pre-existing
synthetic-timestamp test is also a good case study in how
introducing a new default can quietly break tests that relied on the
old unbounded behavior — the author handled it cleanly by swapping
to `sinon.stub(Date, "now")` and a smaller, explicit delay.
