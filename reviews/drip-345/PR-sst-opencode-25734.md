# sst/opencode#25734 — fix(opencode): add max_retries config to cap session retry attempts

- PR ref: `sst/opencode#25734` (https://github.com/sst/opencode/pull/25734)
- Head SHA: `e4cb90e2424b27fc67051cc8e28c427470f1efa9`
- Title: fix(opencode): add max_retries config to cap session retry attempts
- Verdict: **merge-after-nits**

## Review

The schema addition at `packages/opencode/src/config/config.ts:260-263` is appropriately
placed inside the `experimental` block — exactly where you'd want a knob like this to land
while semantics are still being shaken out. Defaulting to "unbounded retries when not set"
is also the right call: it preserves today's behavior, which is what users with no
expectation of this knob should observe.

The wiring in `packages/opencode/src/session/processor.ts:678` reads
`max_retries` once per session iteration and then passes it through to
`SessionRetry.policy({ maxRetries, ... })`. The early-exit in
`packages/opencode/src/session/retry.ts:113` (`meta.attempt >= opts.maxRetries`)
short-circuits the schedule before the delay/wait branch, so the bound is enforced
without paying for an unnecessary backoff sleep on the final attempt. Clean.

Nit: the policy now silently drops the retried error on the floor when the cap is hit
(`Cause.done(meta.attempt)` matches the "not retryable" branch above it). It would help
debuggability to surface a message like `"Max retry attempts (${maxRetries}) exceeded"`
through the existing `set:` callback at `processor.ts:705`, so users see *why* the loop
stopped instead of just "loop ended". Also worth a one-line test in
`packages/opencode/test/session/retry.test.ts` that asserts the policy halts at exactly
`maxRetries` attempts on a perpetually-retryable error — otherwise this branch is
untested.
