# sst/opencode #25586 — fix: propagate chunkTimeout to streamText and add max_retries config

- PR: https://github.com/sst/opencode/pull/25586
- Author: Fatty911
- Head SHA: `54e4958407109ae8f45e16337d78d4874430db72`
- Updated: 2026-05-03T12:51:27Z

## Summary
Adds two related knobs: a per-provider/per-model `chunkTimeout` option propagated into the AI SDK's `streamText` `timeout.chunkMs` to detect silent SSE stalls, and an `experimental.max_retries` schema field. Targets the "stream stays open but no chunks arrive" failure mode.

## Observations
- `packages/opencode/src/config/config.ts` line ~244-269: the entire `experimental` schema block is re-indented one level deeper (4-space inside the parent struct) without a structural reason — this inflates the diff and risks merge churn. The actual semantic addition is just the new `max_retries` field; please revert the cosmetic indentation drift so the diff shows the one-line change only.
- `packages/opencode/src/session/llm.ts` around line ~336: the chunkTimeout lookup walks `options["chunkTimeout"]` then `item.options?.["chunkTimeout"]`. Two issues: (1) negative or zero values fall through the `typeof === "number"` test and would be passed straight to `chunkMs`, which the AI SDK may treat as "fire immediately"; add `chunkTimeout > 0` guard. (2) The schema (`config.ts`) does not declare `chunkTimeout` at all — it is read dynamically from untyped `options`. Either expose it in the schema (alongside `mcp_timeout`) so users get autocomplete and validation, or document why it is intentionally schema-less.
- Same file line ~384: `...(chunkTimeout ? { timeout: { chunkMs: chunkTimeout } } : {})` — good conditional spread, but no companion `max_retries` plumbing is visible in the LLM call site even though `max_retries` was just added to the schema. The PR title promises both; right now `max_retries` looks declared-but-unused. Wire it into the streamText call (`maxRetries: ...`) or split the PR.
- No tests in the diff for the new chunkTimeout abort path. Given this is meant to fix silent dropouts, a unit test that mocks a stalled stream and verifies the abort fires would harden it considerably.

## Verdict
`request-changes`
