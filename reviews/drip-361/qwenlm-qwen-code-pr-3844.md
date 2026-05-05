# QwenLM/qwen-code #3844 — feat(tools): add WebSearch tool with prompt-injection defenses

- **Head SHA:** `ab49448d18678542cd3a1a8eff285fac0d113398`
- **Author:** @wenshao
- **Verdict:** `merge-after-nits`
- **Files:** +785 / −3 across `packages/core/src/tools/web-search.{ts,test.ts}` (new, 460+300 lines), plus 9 wiring files: `config/config.ts`, `extension/claude-converter.ts`, `followup/speculationToolGate.ts`, `index.ts`, `permissions/{permission-manager,rule-parser}.ts`, `services/microcompaction/microcompact.ts`, `tools/{tool-error,tool-names}.ts`

## Rationale

This is path-C from #3841: an explicit `web_search` tool that talks to DashScope's OpenAI-compat `/chat/completions` with `enable_search`/`forced_search`/`enable_source`/`search_options` parameters and parses `search_info` back out. The PR is well-scoped despite its line count — 760 of the 785 added lines are the tool itself + its tests, and the remaining 25 are the 9 thin wiring touches needed to register the tool consistently across naming, permissions, speculation gating, microcompaction, and Claude-tool conversion. All 9 wiring touches are surgical:

- `tools/tool-names.ts:33,63` adds `WEB_SEARCH: 'web_search'` to both `ToolNames` (canonical id) and `ToolDisplayNames` (display-name map) — paired correctly so the tool is discoverable both ways.
- `extension/claude-converter.ts:111` flips `WebSearch: 'None'` → `WebSearch: 'WebSearch'` so Claude Code-format conversations now actually map their `WebSearch` tool calls to the new tool instead of silently dropping them.
- `permissions/permission-manager.ts:441` adds `'web_search'` to the recognized tool name list (necessary for `--allowed-tools web_search` CLI parsing).
- `permissions/rule-parser.ts:96-99` adds the alias triplet (`web_search` / `WebSearch` / `WebSearchTool`) — matches the existing `WebFetch` pattern at the same file.
- `services/microcompaction/microcompact.ts:20` adds `WEB_SEARCH` to `COMPACTABLE_TOOLS` — so search results don't hang around verbatim in long sessions and instead get summarized like web-fetch results already do.
- `followup/speculationToolGate.ts:53` adds `WEB_SEARCH` to `BOUNDARY_TOOLS` (alongside the existing `WEB_FETCH`) — so speculative continuation halts at a search call. The accompanying comment edit at lines 36-38 is a nice touch: it explicitly documents *why* `web_search` and `web_fetch` are *excluded* from `SAFE_READ_ONLY_TOOLS` (network requests + user confirmation) and points the reader to the boundary list — avoids future confusion.
- `tools/tool-error.ts:67-71` adds four new error types (`WEB_SEARCH_RATE_LIMITED`, `WEB_SEARCH_BACKEND_FAILED`, `WEB_SEARCH_NO_RESULTS`, `WEB_SEARCH_PROVIDER_UNSUPPORTED`) — used in the test file so they're exercised end-to-end.
- `index.ts:131` exports `WebSearchTool` and `WebSearchToolParams` types for SDK consumers.
- `config/config.ts:2883-2886` registers the tool via `registerLazy` with a dynamic `import('../tools/web-search.js')` — matches the lazy-load convention used for `WebFetchTool` immediately above (line 2879-2882). Lazy-load is correct here because `web-search.ts` is 460 lines and most sessions won't use it.

The defenses claimed in the PR body all show up in the test file:
- `web-search.test.ts:71-78` rejects `query` < 2 chars and `allowed_domains` > 25 (build-time validation).
- `web-search.test.ts:83-93` confirms `WEB_SEARCH_PROVIDER_UNSUPPORTED` when `apiKey` is missing — and importantly asserts `mockFetch` was *not* called, so the tool fails closed before sending a request.
- `web-search.test.ts:118-148` verifies `assigned_site_list` propagation from `allowed_domains` to the request body and that `enable_search` + `forced_search` are forwarded.
- `web-search.test.ts:150+` (truncated in diff but the imports include the helper) covers `search_info` location compatibility (top-level vs `choices[0].message.search_info`).

**Concrete concerns, in priority order:**

1. **`WeakMap<Config>` rate-limit counter (`max_uses = 8` per session) is not safe across `Config` re-instantiation.** The PR body says "via `WeakMap<Config>`" — that's keyed by `Config` instance identity, not by `getSessionId()`. If anything in the orchestrator constructs a fresh `Config` mid-session (or if hot-reload swaps the instance), the counter resets to 0 and the model can blow past 8. The test file imports `__resetWebSearchCallCount(mockConfig)` so the tool exposes a debug-only reset hook, but the production semantics need to be: "per session id" (sticky across `Config` swaps) or explicitly documented as "per-Config-instance, intended to reset on reload". I'd want one test that verifies the counter survives a no-op `Config` mutation and one comment in `web-search.ts` about the reset semantics.

2. **`MAX_RESULT_SIZE_CHARS = 100_000` is a hard char cap, not a token cap.** For CJK-heavy results that's ~50K tokens, plenty; for English, ~25K tokens, also plenty — but DashScope's search backend returns variable-length snippets, and a 100K-char truncation could cut mid-utf8-codepoint or mid-result. Confirm the truncation slice happens on result-array boundaries, not on the joined-output string.

3. **`blocked_domains` is enforced client-side only (post-response).** That's fine for safety filtering, but worth a one-line comment in the tool that says: "Blocked domains are filtered after the upstream search; they are not forwarded to DashScope. If you need an upstream-enforced block, use `allowed_domains` (which becomes `assigned_site_list`)." Otherwise users will assume `blocked_domains` saves cost/bandwidth, and it doesn't.

4. **Manual verification box is unchecked.** PR body's checklist explicitly leaves "Manual verification against a real DashScope API key" unchecked. The 15 unit tests + 282 affected existing tests + tsc + eslint all green is excellent CI hygiene, but the actual `enable_search`/`forced_search` HTTP contract against a live DashScope endpoint is not exercised by mocks. Pre-merge smoke against a real key would catch any silent contract drift in the DashScope OpenAI-compat shim.

5. **No settings schema yet for `webSearch.enabled` / `provider` / `extraBody`.** PR body lists this as a follow-up. That's fine — but it means right now the tool always registers (config.ts:2883-2886 unconditionally) even on non-DashScope providers. The runtime check at exec time will return `WEB_SEARCH_PROVIDER_UNSUPPORTED`, which is correct, but a user on a non-DashScope provider will still see `web_search` in their tool list, get the model to call it, and only then learn it's unsupported. A simple "registered but disabled" gate — or at minimum a clear error message that names the provider's actual web-search alternative ("use web_fetch with a search URL") — would prevent confused bug reports.

The tool itself is well-defended by intent and the test coverage maps clearly to the threat model. The wiring touches are mechanically clean and consistent with the existing `web_fetch` patterns. Ship after addressing #1 (the rate-limit semantics — needs at minimum a doc-comment, ideally a session-id-keyed counter) and #4 (live DashScope smoke). The other three are doc/follow-up nits.
