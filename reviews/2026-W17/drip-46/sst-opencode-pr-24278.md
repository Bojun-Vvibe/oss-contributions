# sst/opencode #24278 — fix(provider): prevent duplicate OAuth requests to Google when using Vertex AI provider

- **PR:** https://github.com/sst/opencode/pull/24278
- **Head SHA:** `6d89813432ac69ca28430235f241c8424c492523`
- **Files changed:** 1 — `packages/opencode/src/provider/provider.ts` (+38/−24)

## Summary

Closes #24281. Previously the Vertex AI loader caused two `POST https://oauth2.googleapis.com/token` requests per LLM call: opencode's `fetch` wrapper called `google-auth-library` to mint a Bearer header, and the underlying SDK independently called `google-auth-library` again to authenticate its own request. The fix introduces a memoized `authResult` (token + expiry) at provider-init time, populated lazily on the first `fetch`, and reuses the cached token for subsequent calls.

## Line-level call-outs

- `packages/opencode/src/provider/provider.ts:27-37` — `authPromise` is constructed eagerly with `Effect.tryPromise(...)` but only `runPromise`'d on first `fetch`. Three concrete problems:
  1. **No expiry check.** `authResult.expiresAt` is captured but never consulted on subsequent calls. Google access tokens are 1-hour TTL; a long-lived opencode session will keep using a stale token until the API starts returning 401, at which point this code has no refresh path. The whole point of caching the `expiresAt` field is to compare it against `Date.now()` — that comparison is missing.
  2. **`Effect.catch(() => Effect.succeed(null))`** swallows the auth failure into a `null` result. The `:53-58` block then throws `Error("Google Vertex AI auth failed: no valid access token")` with no underlying cause attached, so the user sees a generic message instead of "ADC not found" / "service account file unreadable" / etc. Re-raise the original error or attach it as `cause`.
  3. **Concurrency.** Two near-simultaneous LLM calls on a fresh session both see `authResult === null` and both call `Effect.runPromise(authPromise)`. Because `authPromise` is an Effect (not a memoized JS Promise), each `runPromise` re-executes it, defeating the dedup. Wrap with `Effect.cached` / `Effect.memoize` or use a plain JS `Promise` with the standard "in-flight promise" idiom. As written this PR reduces the duplicate-OAuth count from 2× per call to "still N× on a cold-start burst", not to 1.
- `:60-65` — `headers.set("Authorization", "Bearer ${authResult.token}")` removed the previous behaviour where the SDK *also* set its own auth header. This is the "SDK now provides the Bearer token directly" behaviour the PR description claims, but I don't see the corresponding SDK option being passed (e.g. `googleAuthOptions: { credentials: { type: 'authenticated_user' } }` or similar). It works because the SDK falls back to whatever's already on the headers, but if the SDK ever decides to overwrite `Authorization` on its own auth path, this silently breaks. A test asserting "exactly one outbound auth request per LLM call" would catch that regression — currently absent.
- `:430` — the `Effect.fnUntraced(...)` → `Effect.fn("Provider.google-vertex")(...)` rename is purely a tracing-name change and is a fine drive-by.
- Whitespace-only churn at `:14-24`, `:67-78`, `:90`, `:98-114`, and `:120-126` — this is autoformatter drift from a different Prettier/dprint config and unrelated to the OAuth fix. It bloats the diff from `~12 lines of substance` to `+38/−24` and forces reviewers to wade through reflowed object literals to find the actual change. Strongly recommend reverting all whitespace-only hunks and resubmitting the substantive change in isolation.

## Verdict

**request-changes**

## Rationale

The intent is right and the cache is the correct first move, but the implementation has three real bugs (no TTL check, swallowed cause, no concurrency dedup) and the diff is polluted with formatter churn. The TTL gap alone will cause silent hour-mark failures in long sessions — the symptom is "Vertex calls start 401-ing 60 minutes in and never recover until restart". Fix the three issues, drop the whitespace, and add either an expiry-respecting test or a single-flight assertion, and this becomes merge-as-is. Until then this is a partial fix that risks regressing into a worse failure mode than the duplicate-request bug it set out to solve.
