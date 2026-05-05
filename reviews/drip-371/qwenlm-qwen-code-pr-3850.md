# QwenLM/qwen-code PR #3850 — refactor(core): classify retry errors

- URL: https://github.com/QwenLM/qwen-code/pull/3850
- Head SHA: `09a62b2f2f6e5311b400a2d25fb153cb385e9e44`
- Author: yiliang114 (易良) — DRAFT
- Files: new `packages/core/src/utils/retryErrorClassification.{ts,test.ts}` + integrations into `retry.ts` and `core/geminiChat.ts` (+527/-5)

## Assessment

This PR introduces a centralized `classifyRetryError(error, { authType })` helper that produces a stable structured classification (`kind`, `decision`, `reason`, `statusCode`, `providerCode`, `providerMessage`, `requestId`, transport fields) so downstream retry logic can make fail-fast vs. fallback decisions from explicit categorization instead of re-parsing status codes at every call site. As the second step in a stacked PR series after the retry-delay-policy work, this is the right factoring: classification logic in one module, with 244 lines of unit tests that pin specific real-world error shapes (HTTP 429/503/529/400/401/500, provider codes 1302/1305, SSE-embedded `HTTP_STATUS/429`, `Throttling.AllocationQuota`, Qwen OAuth `insufficient_quota`, `ETIMEDOUT`, generic SDK `ERR_BAD_REQUEST`, abort errors, invalid status fields).

The classification boundaries called out by the author in the PR summary are the right ones to scrutinize. The test at `retryErrorClassification.test.ts:19-27` correctly classifies HTTP 503 as `decision: 'retryable', reason: 'rate-limit'` — that aligns with the existing stream-retry semantics where 503 ("Provider overloaded") is treated equivalently to 429 because providers like Gemini and Qwen return either depending on the overload type. The provider-code mapping at `:39-63` correctly identifies both string `"1302"` (parsed from JSON-encoded error body) and integer `1305` (parsed from object-shaped error) as the same retryable rate-limit class. The SSE-embedded form at `:67-79` is the most delicate case: the parser correctly extracts `HTTP_STATUS/429`, `code: "Throttling.RateLimit"`, `message`, and `request_id` from a multi-line `id:1\nevent:error\n:HTTP_STATUS/429\ndata:{...}` payload — that's the exact wire format Qwen's streaming endpoint emits for mid-stream rate limits, so getting `requestId: "req-1"` plumbed through means downstream telemetry can correlate retries with upstream traces.

Integration into `retry.ts:201-205` and `:266-340` is non-invasive: `classifyRetryError` is invoked once per attempt and threaded into the `debugLogger.warn(message, retryClassification, error)` calls without changing the retry control flow. Same in `geminiChat.ts:447-465` and `:481-497` where the classification fields (`classificationDecision`, `errorKind`, `classificationReason`) are added to the existing `Rate limit retry scheduled` and `Rate limit retry exhausted` log structures. The control flow stays identical, only observability improves — that's the right shape for a stacked refactor that lets later PRs flip behavior on the new fields.

Concerns: (1) `package-lock.json` has a stray `"peer": true` removal on a `darwin`-only package (`fsevents`?) — unrelated to this PR's scope and should be reverted to keep the diff focused; (2) the reviewer-focus list in the PR is large (HTTP, SSE, provider codes, Qwen OAuth quota, transport, abort) which is appropriate for a 527-line refactor but argues for splitting into "introduce classifier" + "wire into retry.ts" + "wire into geminiChat.ts" sub-PRs for easier review. Status is DRAFT so the author likely intends polish before mark-ready. The classifier itself looks well-structured and the test coverage is the strongest signal here.

## Verdict

`merge-after-nits`
