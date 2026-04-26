---
pr: 10384
repo: cline/cline
sha: 65b1c5e2993f0cc7b8e3d1cc9b06300d57120b3e
verdict: request-changes
date: 2026-04-26
---

# cline/cline #10384 — fix: cap retry-after delay to prevent silent multi-hour hangs

- **URL**: https://github.com/cline/cline/pull/10384
- **Author**: NgoQuocViet2001
- **Head SHA**: 65b1c5e2993f0cc7b8e3d1cc9b06300d57120b3e
- **Size**: +68/-7 across 2 files (`src/core/api/retry.ts` + `src/core/api/retry.test.ts`)
- **Fixes**: #10139

## Scope

Adds `maxRetryAfter` (default `60_000`ms) to the `withRetry` decorator's options. When a `retry-after` header produces a delay greater than the cap, the decorator throws the original error immediately instead of waiting. Two new tests cover the throw-immediately and the within-cap retry paths. The existing Unix-timestamp test is reworked to stub `Date.now` and pass `maxRetryAfter: Infinity` so its synthetic 30-year-future timestamp doesn't trip the new cap.

## Specific findings

- **Default of 60s is reasonable for the user-facing problem** (Gemini 3-hour `retry-after` causing silent hangs, per #10139). Most legitimate `retry-after` from production providers is 1-30 seconds; 60s covers the OpenAI/Anthropic burst-quota recovery window.

- **Behavior contract changes silently for anything passing `retry-after > 60s`.** This is the key concern: the decorator previously *retried* on anything 429 with a header; it now *throws immediately* if the delay exceeds the new default. Any caller that used `@withRetry` without overriding `maxRetryAfter` and was *relying* on the long-wait behavior (e.g. an overnight batch job that's fine to wait an hour for a quota window) is now broken without warning. The PR doesn't enumerate which decorated methods are affected. A `git grep "@withRetry"` across `src/api/providers/*.ts` would show the blast radius — that should be in the PR description.

- **Reworked Unix-timestamp test at `retry.test.ts:114-145` introduces a hidden problem.** The new test stubs `Date.now()` with `sinon.stub(Date, "now").returns(fakeNow)` but never restores it in an `afterEach`. If any test in the file (or downstream of this file in mocha's run order) depends on real wall-clock time, it will silently see `fakeNow = 1700000000000` (= 2023-11-14). I'd expect this to break the existing exponential-backoff or maxDelay tests below it (e.g. `should respect maxDelay` at `:198`). Author should:
  - Use `let dateNowStub` + `afterEach(() => dateNowStub?.restore())`, or
  - Use `sinon.useFakeTimers()` with proper teardown, or
  - Move the `Date.now` stub into a `beforeEach`/`afterEach` pair scoped to that one `describe` block.

- **The `delay > maxRetryAfter` check at `retry.ts:69` runs *before* the existing `delay = Math.min(maxDelay, ...)` cap that exists for the no-header branch.** Read carefully: the current PR only applies the cap inside the `if (retryAfter)` branch; the no-header exponential-backoff path still uses `maxDelay` (10s default). That's *probably* intentional (header values are operator-trusted; no-header backoff is decorator-controlled), but it means there are now two separate caps with different semantics and defaults. Worth adding a one-line comment at `retry.ts:67` explaining this asymmetry, otherwise a future maintainer will "consolidate" them and break header-honoring behavior.

- **The `throw error` on cap-exceed at `retry.ts:70` re-throws the *original* 429 error.** Good — the caller sees the same error shape they'd see if there was no decorator at all, and the `error.headers["retry-after"]` is still on it for surface-level UI to display "retry in N seconds" rather than silently hanging. But there's no log breadcrumb at the throw site. Add `console.warn(...)` or whatever logger this codebase uses ("retry-after delay Nms exceeds cap Nms; surfacing error") so operations can correlate user-visible 429s with the new cap behavior.

- **`Number.parseInt` substitution at `retry.ts:60`** is an unrelated lint-style change. Functionally identical in this context; mention in the PR description so the diff isn't surprising. Same for `status: number = 429` → `status = 429` at `retry.ts:35`. These are biome auto-fixes; reasonable to include but should be called out.

- **Test gap: no test for the case where `retry-after` is exactly `maxRetryAfter`.** The `>` boundary at `:69` means `delay === maxRetryAfter` retries; `delay === maxRetryAfter + 1` throws. Add a test at the boundary; cheap insurance against a future `>=` typo.

- **No test for the Unix-timestamp form crossing the cap.** The two new tests use the delta-seconds form (`"10800"` and `"0.01"`). The Unix-timestamp branch at `retry.ts:62-63` produces the same `delay` variable so the cap check applies there too, but a test asserting "Unix timestamp 3 hours in the future also throws" would be the obvious symmetry.

## Risk

Medium-high — *not* because the fix is wrong, but because the default-60s cap is a behavior change for every existing `@withRetry` call site, and the PR doesn't audit which call sites are affected. The Date.now-stub-without-teardown in the test file is a flake-shaped landmine. Both should be addressed before merge.

## Required changes

1. **Restore `Date.now` after the stub** in `retry.test.ts:115-117` (use `sinon.restore()` in an `afterEach` or scope-stub via `beforeEach`/`afterEach`).
2. **Audit `@withRetry` call sites** and either:
   - Confirm none of them need >60s retry-after honoring, OR
   - Add per-site `maxRetryAfter` overrides where long waits are intentional, OR
   - Make the default `Infinity` (preserve old behavior) and require call sites to opt in to the cap.
3. **Document the decision** in the PR body so reviewers and future maintainers understand the blast radius.

## Nits

4. Add a boundary test (`delay === maxRetryAfter`).
5. Add a Unix-timestamp + cap-exceeded test (symmetry with delta-seconds case).
6. Add a logger breadcrumb at the throw site so cap-triggered errors are observable.
7. Comment the asymmetry between `maxRetryAfter` (header path) and `maxDelay` (no-header path) at `retry.ts:67`.
8. Call out the unrelated biome auto-fixes in the PR description.

## Verdict

**request-changes** — the symptom-fix is correct and overdue (multi-hour silent hangs are unacceptable), but the test-cleanup hazard and the unaudited blast-radius on default behavior change need to be resolved before this is safe to land. Fixable in a single follow-up commit.

## What I learned

When a decorator's default behavior changes from "honor whatever the server says" to "honor up to a cap, then throw", the change is API-shape-additive (new option) but behavior-breaking for every undecorated callsite. The migration-friendly path is `default: Infinity` + opt-in cap; the user-friendly path is `default: 60_000` + audit + per-site override. Both are defensible, but picking the second without doing the audit creates a hidden-regression risk that's hard to attribute later (the symptom is "things stopped retrying that used to retry"; root cause is one line in a decorator file no one looks at). The test-side `Date.now` stub-without-restore is also worth filing as a reusable lint: any `sinon.stub(Date, ...)` outside a `beforeEach`/`afterEach` pair is a leaky-time bug waiting to happen.
