# Review: QwenLM/qwen-code #3810 — Clear FileReadCache on every history rewrite path

- PR: https://github.com/QwenLM/qwen-code/pull/3810
- Head SHA: `0bbee541d9cd497beb8230e0cdc5df91f93b5ab9`
- Author: wenshao
- Files touched: `packages/core/src/core/client.test.ts`, `packages/core/src/core/client.ts`

## Verdict: `merge-as-is`

## Rationale

The change identifies five history-rewrite paths in `GeminiClient` that could leave `FileReadCache` populated with entries pointing at content the model has just been told it never saw, then plumbs `getFileReadCache().clear()` into each: (1) `resetChat` (test at `client.test.ts:460-469`), (2) `setHistory` (test at `client.test.ts:475-485`), (3) `truncateHistory` (test at `client.test.ts:487-498`), (4) `stripOrphanedUserEntriesFromHistory` retry path (test at `client.test.ts:500-528`), and (5) the microcompaction branch in `sendMessageStream` that strips old `read_file` results when the idle gap exceeds `toolResultsThresholdMinutes` (test at `client.test.ts:580-642`). The microcompaction test is particularly thoughtful — it constructs 6 `read_file` interaction pairs and forces the timestamp 90 minutes back so the if-meta branch fires, then asserts both `setHistory` was called *and* the cache was cleared, and pairs it with a negative test (`client.test.ts:625-642`) where the idle gap is 30 seconds and the cache stays untouched. The justification comment in `client.ts:208-209` explains the failure mode crisply ("Stripped trailing user entries can include read_file functionResponses from a failed-then-retried request"). No nits — coverage and reasoning are both above the bar.