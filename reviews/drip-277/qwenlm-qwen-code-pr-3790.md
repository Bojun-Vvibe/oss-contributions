# Review — QwenLM/qwen-code#3790

- PR: https://github.com/QwenLM/qwen-code/pull/3790
- Title: fix(core): improve stream rate-limit retry diagnostics
- Head SHA: `69c9be5475803088dab72178e3b534012c746ba9`
- Size: +567 / −10 across 4 files
- Verdict: **merge-after-nits**

## Summary

Two coupled improvements to the streamed rate-limit retry path in
`packages/core/src/core/geminiChat.ts`. (1) The previously fixed
60s delay between retries becomes exponential
(`initialDelayMs * 2^(attempt-1)`) capped at 5 minutes, with
`Retry-After` honoured as a provider-supplied minimum but still
clamped by `maxDelayMs` so a misbehaving header can't park the TUI
indefinitely. (2) A new `getRateLimitErrorDetails(error)` helper
extracts structured diagnostic fields (statusCode, providerCode,
providerMessage, requestId, transport) from both HTTP and SSE error
shapes; these get logged on every retry/exhaustion event for
observability without changing retry decisions.

## Evidence

- `packages/core/src/core/geminiChat.ts:108-115` reshapes
  `RATE_LIMIT_RETRY_OPTIONS` from `{ maxRetries, delayMs: 60000 }`
  to `{ maxRetries, initialDelayMs: 60000, maxDelayMs: 300000 }`.
- Same file, lines 433-453 in the stream-side retry block: the old
  `const delayMs = RATE_LIMIT_RETRY_OPTIONS.delayMs` is replaced by
  `getRateLimitRetryDelayMs(rateLimitRetryCount, { ...options,
  error })` which folds in the new exponential + Retry-After
  logic. The structured `debugLogger.warn('Rate limit retry
  scheduled', { retryPath: 'stream', retryDecision: 'retry',
  attempt, retryDelayMs, ...details })` replaces the old human-
  readable string.
- `packages/core/src/utils/rateLimit.ts:436-509` adds
  `RateLimitErrorDetails`, `RateLimitRetryDelayOptions`,
  `getRateLimitErrorDetails`, and `getRateLimitRetryDelayMs`.
  Transport detection keys on `event:error` / `HTTP_STATUS/`
  substrings to flag SSE-style errors vs HTTP errors.
- Test coverage in `packages/core/src/core/geminiChat.test.ts:1304`
  (`should use Retry-After delay for streamed rate-limit errors`)
  asserts `delayMs === 180_000` when a 429 carries `retry-after:
  180`. Lines 1619-1700 (`should increase delay across repeated
  streamed rate-limit errors`) verify the geometric progression
  `[60_000, 120_000]` across two attempts.

## Notes / nits

- `getRateLimitRetryDelayMs` uses `Math.pow(2, attempt-1)` without
  jitter. Two TUIs sharing the same DashScope quota that get
  throttled in the same second will retry in lockstep — a small
  ±10% jitter (`delayMs * (0.9 + Math.random() * 0.2)`) would
  spread the herd. Not a blocker for a single-user CLI but cheap
  to add.
- Transport detection by substring (`event:error`,
  `HTTP_STATUS/`) is fragile. A future SSE provider that uses
  different framing will fall through to `transport: 'unknown'`.
  Acceptable for a debug field, but worth a TODO calling out that
  this is heuristic.
- The doc comment on `RATE_LIMIT_RETRY_OPTIONS` still mentions
  Claude Code parity for `maxRetries: 10`. Combined with `5min`
  cap and exponential growth from 60s, the worst-case wall time is
  60+120+240+300+300+300+300+300+300+300 ≈ 42 minutes — that's a
  long time for an interactive CLI to sit blocked. Consider
  reducing `maxRetries` or surfacing a per-attempt user prompt
  after, say, attempt 3.
- The new `getRateLimitErrorDetails` helper sits next to its tests
  (`rateLimit.test.ts` grew significantly per the file count) so
  the pure-function piece is well-covered; nothing further needed
  there.

Ship after the jitter and worst-case-wall-time decisions.
