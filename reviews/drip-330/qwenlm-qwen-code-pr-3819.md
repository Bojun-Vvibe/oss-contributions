# QwenLM/qwen-code #3819 — fix(core): prevent duplicate MCP processes from concurrent discovery

- SHA: `6ab6703a890b339abdabd4960dfe79ad6943ae2b`
- State: OPEN, +997/-41 across 6 files
- Closes: #3817

## Summary

Adds an in-flight discovery guard (`Set<string>`) to `McpClientManager.discoverMcpToolsForServer()` so concurrent re-discovery requests for the same server (health-check reconnect racing with incremental discovery) coalesce into one connect, preventing the duplicate-MCP-process bug evidenced in the issue (PIDs 30477 + 30513 spawned 21s apart for the same server). Also adds three retry-policy expansions (408, transient 409, ECONNRESET) in `geminiChat.ts` + `retry.ts`.

## Notes

- **PR-scope concern (significant)**: The PR title / body describe a focused MCP fix (~120 lines across `mcp-client-manager.ts` + its test). The actual diff bundles **+692/-37** in `retry.ts` + `geminiChat.ts` + `retry.test.ts` + `geminiChat.test.ts` (+167/-10 retry source, +525/-27 retry tests, +12/-4 chat source, +174 chat tests). These are two unrelated bug fixes. Strongly recommend splitting into two PRs so review and bisect are tractable. If kept together, retitle to "fix(core): MCP concurrent-discovery guard + retry coverage for 408/409/ECONNRESET".
- `packages/core/src/tools/mcp-client-manager.ts:+23/-0` — the load-bearing change. From the PR body: adds `inFlightDiscoveries: Set<string>`, early-returns if already in flight (with debug log), cleans up failed clients in catch, stops health-check timer during re-discovery disconnect, and clears the set in `stop()`. The early-return-on-concurrent shape is correct; the only question is whether returning *immediately* (vs awaiting the in-flight discovery's completion) is the right semantic — a caller that returns before tools are listed sees an empty tool set for one tick. If health-check is the only secondary caller, that's fine; if anything else relies on "after this returns, tools are ready", they break. Worth confirming in the test.
- `packages/core/src/tools/mcp-client-manager.test.ts:+96` — two new tests per PR body: (1) concurrent-dedup (two simultaneous calls, slow connect, assert one connect); (2) in-flight cleanup (after success, second call proceeds). The cleanup test is critical — without it, a one-time discovery would lock the server permanently. Good. Missing: a *failure* cleanup test that asserts `inFlightDiscoveries` is cleared on error so subsequent retries can proceed. The PR body says "Clean up failed clients in catch block" but a unit test pinning that behavior is essential.
- `packages/core/src/core/geminiChat.ts:18-22` — pulls in `classifyError` from `retry.ts`. Used at the call site (not in this slice) to decide retry vs. rethrow. Cleaner than inlining status-code checks.
- `packages/core/src/core/geminiChat.test.ts:1700+` — four new test groups covering 408 retry, transient 409 retry, deterministic 409 *no* retry, and ECONNRESET retry. The transient/deterministic split for 409 (lines 1746-1839 region) is the right call — `Lock contention detected` retries, `Resource already exists` doesn't. The classifier in `retry.ts` must inspect message text to make this distinction; brittle but unavoidable for vendors that overload 409. Worth one comment in `retry.ts` about which substrings are matched, otherwise future contributors will silently flip behavior by tweaking copy.
- `packages/core/src/utils/retry.ts:+167/-10` — substantial reshape of the retry classifier. Not in this slice. Reviewer should confirm the new shape preserves existing exponential-backoff tunables and doesn't accidentally retry on idempotency-violation errors (4xx that aren't transient).
- `packages/core/src/utils/retry.test.ts:+525/-27` — large test expansion. The +525 line growth on a previously-stable file is by itself a reason to land in two PRs.
- Risk statement in PR body ("Low. Only affects the discoverMcpToolsForServer path") is true *only* of the MCP slice. The retry changes touch every model call. The PR body does not assess that risk.

## Verdict

`request-changes` — the MCP concurrent-discovery guard itself is well-conceived and well-tested and would be a clean `merge-after-nits` on its own. The bundled retry rework is unrelated, materially expands risk, and is mis-scoped relative to the PR title. Split into two PRs: (a) MCP guard with a failure-path cleanup test added; (b) retry-policy expansion with its own risk discussion. If maintainers prefer to keep bundled, at minimum retitle and add the missing failure-cleanup test for `inFlightDiscoveries`.
