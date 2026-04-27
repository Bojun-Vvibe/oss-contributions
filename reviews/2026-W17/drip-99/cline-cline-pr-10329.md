# cline/cline#10329 — feat(cost-control): enforce third-party API spend limits

- **Head**: `8588750283031f069ef66157ce5c798bb9fe98d7`
- **Size**: +1000/-11 across 17 files
- **Verdict**: `merge-after-nits`

## Context

Extends the existing first-party (Cline-provider) spend-limit enforcement
to third-party providers (Anthropic, OpenAI, Gemini, etc.) used through
the BYO-key path. The enterprise constraint: orgs that set a daily or
monthly USD budget cap on member spend want that cap enforced no matter
which model the user picks, not just on Cline-provider routes. Current
behavior leaks: a user over budget on Cline-provider routes can still
burn unlimited dollars through their personal Anthropic key.

The PR introduces a `ThirdPartySpendLimitService` (cached overbudget
status), warms the cache on auth restore, gates outgoing requests in
`Task.checkThirdPartySpendLimit`, and reuses the existing
`SpendLimitError` UI by throwing a `ClineError` of a new
`ClineErrorType.SpendLimit` shape. Plus a mock server, a new RPC for
"Request Increase", and tests.

## Design analysis

### Error-classification gating: `ClineError.ts:8, 145-150`

```ts
+  SpendLimit = "spendLimit",
...
+  // Check spend limit exceeded (org-enforced budget cap, 429 SPEND_LIMIT_EXCEEDED)
+  // Must be checked before the generic rate-limit check since both use 429
+  if (code === "SPEND_LIMIT_EXCEEDED" || details?.code === "SPEND_LIMIT_EXCEEDED") {
+    return ClineErrorType.SpendLimit
+  }
```

Order is correct and the comment captures *why* — both classes return
HTTP 429, so without an explicit code check, the existing rate-limit
arm would absorb spend-limit errors and trigger the 3-attempt
exponential-backoff retry. That would burn through the org's budget
even faster (3 retries at 2s/4s/8s = up to 3× the dollars you tried to
prevent). Putting the spend-limit check first is the right ordering.

### Retry-loop gating: `task/index.ts:2148-2207`

Three call sites correctly thread through `isSpendLimitError`:

```ts
const isSpendLimitError = clineError.isErrorType(ClineErrorType.SpendLimit)
...
if (
  !isClineProviderInsufficientCredits &&
  !isAuthError &&
  !isSpendLimitError &&
  this.taskState.autoRetryAttempts < 3
) {
  // ... retry
}
```

And in the streaming-failure path at `:3058-3086`:

```ts
const isStreamingSpendLimitError = clineError.isErrorType(ClineErrorType.SpendLimit)
if (!isStreamingSpendLimitError && this.taskState.autoRetryAttempts < 3) {
  // ... retry
}
```

Symmetric gating across both retry paths. The "show error_retry with
failed flag" branch at `:2201` is also correctly suppressed for spend-limit
errors — the SpendLimitError UI is the user-facing surface, not the
generic retry-failed banner. ✓

### Pre-flight check: `task/index.ts:1916-1917`

```ts
+ // Block overbudget third-party requests before they reach the provider.
+ await this.checkThirdPartySpendLimit(providerInfo.providerId)
```

The check happens after `getCurrentProviderInfo()` and before the call
to `host.getHostVersion()` — i.e. before any network request to the
third-party provider. Correct: failing locally means the user's API key
doesn't get called at all when they're over budget, and the cached
status path never adds latency to a non-overbudget request.

The implementation at `:1745-1789` (visible in the diff) is well-shaped:

- Skip if `providerId === "cline"` — Cline-provider has server-side
  enforcement, no need to double-check client-side. ✓
- `await svc.fetchIfNeeded()` — uses cached status, only refetches when
  TTL expired (the `ThirdPartySpendLimitService` caches with a TTL).
- `if (!status?.overbudget) return;` — the safe-default branch when
  status is missing (non-enterprise orgs, transient fetch failure) is
  to allow the request, which matches the principle of least
  intrusion.
- Daily takes precedence over monthly when both are set, with a
  comment explaining "it will reset sooner" — that's the right
  user-facing precedence.
- Throws via `ClineError.transform({status: 429, code: "SPEND_LIMIT_EXCEEDED", ...})`
  which routes through the new error-classification path and ends up
  rendered by the existing `SpendLimitError` UI.

### Cache-warm on login: `AuthService.ts:316-326`

```ts
+ // Warm the third-party spend-limit cache so the status is ready
+ // before the user's first task. Fire-and-forget; failures are
+ // fully handled inside the service (logged, not thrown).
+ // Dynamic import to avoid a hard dependency cycle with task layer.
+ import("../spend-limit/ThirdPartySpendLimitService")
+   .then(({ ThirdPartySpendLimitService }) => ThirdPartySpendLimitService.getInstance().fetchIfNeeded())
+   .catch((err) => Logger.debug(`[SpendControl] Cache-warm on login failed: ${err}`))
```

The dynamic-import + fire-and-forget shape is correct for two reasons:

1. **Dynamic import breaks the dependency cycle** — `AuthService` shouldn't
   statically depend on a service that depends on a service that
   ultimately needs auth. Lazy import at the call site sidesteps that.
2. **`.catch(...)` swallows errors** — cache-warm should never fail
   login. Failures get debug-logged; the next pre-flight check will
   re-attempt the fetch.

The trade-off: the first request after login may still pay the
fetch latency if the warm hasn't completed. Acceptable.

### Account-service integration: `ClineAccountService.ts:240-274`

```ts
async fetchOverbudgetStatusRPC(organizationId: string): Promise<OverbudgetStatus | undefined> {
  try {
    return await this.authenticatedRequest<OverbudgetStatus>(`/api/v1/organizations/${organizationId}/budget/overbudget`)
  } catch (error) {
    // 403/404 = non-enterprise org, expected for most users
    if (axios.isAxiosError(error)) {
      const status = error.response?.status
      if (status === 403 || status === 404) {
        return undefined
      }
    }
    Logger.error("Failed to fetch overbudget status (RPC):", error)
    return undefined
  }
}
```

The 403/404-as-feature-not-enabled handling is the right shape — most
users aren't on enterprise orgs, so calling this endpoint will
predictably return 403/404, and treating that as "no spend control,
move on" rather than logging an error keeps the logs clean. Other
errors (5xx, network) get logged + return `undefined`, which the
caller treats as "fail open" (no enforcement). That's the safe default
for a feature that gates legitimate user work — better to under-enforce
than to brick the tool because the budget endpoint is flapping.

### Mock server: `scripts/mock-spend-limit-server.mjs`

97-line dev tool that intercepts `/chat/completions` to return 429
SPEND_LIMIT_EXCEEDED and `/budget/request` to return 204, proxying
everything else to `https://api.cline.bot`. This is *exactly* the right
kind of dev artifact to ship with a feature like this — the SpendLimit
UI is hard to hit in normal dev (you'd have to actually exhaust a real
budget). Bundling a mock server in `scripts/` makes hands-on QA
trivial.

The `REAL_BACKEND` const is hardcoded to `https://api.cline.bot`. For
self-hosted/staging environments this should be env-overridable
(`process.env.CLINE_API_BACKEND ?? "https://api.cline.bot"`). One-line
fix; minor.

## Concerns / nits

1. **Cache TTL not visible in the diff**. `fetchIfNeeded()` is mentioned
   but the TTL constant (`MAX_CACHE_AGE_MS` or similar) isn't in the
   visible hunks. If the cache lives too long (say 5+ minutes), an org
   that bumps a user's limit via the admin UI will have the user still
   blocked client-side until cache expiry. If too short (10s), every
   tool call adds a network round-trip. Worth a documented value
   somewhere visible — body or in the `ThirdPartySpendLimitService.ts`
   header comment.

2. **No invalidation on "Request Increase" success.** The user clicks
   "Request Increase", the org admin approves, the budget bumps —
   but the user's local cache still says "overbudget". Should the
   `submitLimitIncreaseRequestRPC` success path also call
   `ThirdPartySpendLimitService.invalidate()` so the next request
   re-fetches? Or is the org-admin-approves-budget signal expected to
   be polled separately? Worth clarifying.

3. **Daily-vs-monthly precedence comment** says "it will reset sooner".
   Correct for daily-vs-monthly, but if both daily and monthly are
   exceeded, the user really hits *both* — the message text only
   reports daily, which is fine, but the reset-time hint to the user
   might be misleading if monthly resets first (e.g. last day of the
   month, daily resets in 8h but monthly in 6h). Edge case; mention in
   user-facing copy or a follow-up.

4. **`mock-spend-limit-server.mjs:23` hardcodes `REAL_BACKEND = "https://api.cline.bot"`**.
   Make it env-overridable.

5. **Test coverage for the streaming-failure path** at `:3058-3086` is
   not visible in the diff. The non-streaming retry path likely has
   the existing tests covering it; adding one for the streaming path
   asserting "spend-limit error during stream does NOT retry" closes
   the gap.

6. **`ClineError.transform` is being asked to construct a synthetic 429**
   from a local check, which means the wire-shape this UI expects has
   to be matched exactly. If the server-side 429 schema ever evolves
   (`limit_usd` → `limit_cents`, etc.), this client-synthesized version
   will silently drift. Worth a shared schema helper that both server
   and client consume.

## Verdict reasoning

The architecture is right: cache-on-auth-warm, gate-on-pre-flight, reuse
existing UI via shared error type, fire-and-forget on the cache-warm to
avoid blocking login. All three retry-skip points (non-streaming
backoff, retry-failed banner, streaming auto-retry) correctly thread
the new `SpendLimit` error type. The classification ordering (spend-limit
before generic rate-limit) is critical and correctly placed first. Verdict
is `merge-after-nits` solely on the cache-TTL/invalidation visibility
and the streaming-path test gap — both of which are easy follow-ups.

## What I learned

The "two failure modes that share an HTTP status code" trap (429 for
both rate-limit and budget-cap) is a recurring source of silent
misbehavior in retry loops. Server contracts that mint distinct error
codes (`SPEND_LIMIT_EXCEEDED` vs the implicit "rate limit") and clients
that route on the code rather than the status are how you avoid the
"3 retries × $X-per-retry = quintupling the budget overrun" anti-pattern.
The mock-server-in-`scripts/` is also a good template: any feature that
gates real user work on a remote enforcement decision should ship its
own mock so the UI can be exercised without burning real dollars.
