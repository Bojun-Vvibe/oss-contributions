# QwenLM/qwen-code PR #3798 — feat(core): classify retryable transport/provider failures vs deterministic request errors

- **Repo:** QwenLM/qwen-code
- **PR:** #3798
- **Head SHA:** `42ae93ca693f5a630a999668b80590f8fe36ddb8`
- **Author:** B-A-M-N
- **Title:** feat(core): classify retryable transport/provider failures vs deterministic request errors
- **Diff size:** +397 / -4 across 2 files
- **Drip:** drip-294

## Files changed

- `packages/core/src/utils/retry.ts` (+168/-4) — adds two exported helpers: `isRetryableNetworkError(error)` which checks `NodeJS.ErrnoException.code` against a fixed set (`ECONNRESET`, `ETIMEDOUT`, `ESOCKETTIMEDOUT`, `ECONNREFUSED`, `ENOTFOUND`, `EHOSTUNREACH`, `EAI_AGAIN`) plus case-insensitive substring match against the error message ("socket closed", "stream ended", "network error", "connection reset", "econnreset", "etimedout"); and `classifyError(error): { retryable: boolean, reason: string, status?: number }` which buckets HTTP statuses (400/401/403/404/422 → non-retryable, 408/429/5xx → retryable, 409 → retryable iff message contains "lock"/"conflict"/"contention", otherwise deterministic).
- `packages/core/src/utils/retry.test.ts` (+229/-0) — comprehensive `describe` blocks for both helpers covering each errno code, each substring path, each HTTP bucket, and the 409 split-personality cases.

## Specific observations

- Both helpers are pure and exported — easy to compose into existing retry logic without touching the orchestration. Good API shape.
- `retry.ts:isRetryableNetworkError` errno set is reasonable but conspicuously missing `EPIPE` and `ENETUNREACH`, both of which can hit during long-running streaming responses. Worth adding.
- The substring-match path (`error.message.toLowerCase().includes("socket closed")` etc.) is a fragile last-resort fallback for cases where the underlying library doesn't surface an errno code. Fine as a heuristic but the test at `retry.test.ts:1098-1101` (`'ECONNRESET: econnreset'` matches) shows how easy it is for the substring to spuriously catch unrelated errors that just happen to mention a code in their text. Document the substring path as "best-effort, prefer `code`-based classification when possible".
- `retry.ts:classifyError` 409 handling: retries when `message` contains `lock` / `conflict` / `contention` (case-insensitive). The test at `retry.test.ts:1170-1186` exercises both the retry and non-retry branches. Subtle: a 409 with message `"Version conflict"` is *non-retryable* (test at line ~1180) — but the substring check at line 1176 says "should classify 409 with conflict message as retryable" with message `"Conflict detected"`. So the rule is "any occurrence of the substring `conflict`" → retryable. `Version conflict` contains `conflict`, so by the substring rule it should *also* be retryable, contradicting the line ~1180 assertion. Either the production code uses a more specific check (e.g. word-boundary regex) than the diff suggests, or the test is wrong. **Reviewer must inspect the actual `classifyError` implementation in `retry.ts` to resolve this** — the diff snippet I have visibility into doesn't include those lines.
- No integration with the existing `retryWithBackoff(...)` orchestrator visible in this diff. So this PR adds the *vocabulary* but not the *consumer*. That's a fine shape — landing helpers + tests in one PR and the wiring in a follow-up — but the PR description should make this explicit so reviewers don't expect end-to-end behavior change.
- `getErrorStatus` is reused (imported from `./errors.js`) which is exactly right — no duplication of HTTP-status extraction logic.
- Test count is high (~50 distinct cases) but each is shallow. Consider parameterized tables to compress; current shape is readable but verbose.

## Verdict: `merge-after-nits`

Solid foundation for a structured retry policy. Ask: (1) resolve the apparent 409-substring-vs-test contradiction (likely needs a word-boundary check, not `String.includes`), (2) add `EPIPE` and `ENETUNREACH` to the errno set, (3) document the substring-match path as best-effort, (4) PR description should call out that this adds helpers without wiring them into `retryWithBackoff` yet.
