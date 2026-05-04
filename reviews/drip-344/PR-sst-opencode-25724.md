# sst/opencode#25724 — fix: prefill rejection on Claude Opus 4.6+

- PR ref: `sst/opencode#25724`
- Head SHA: `912db73b648dd596d2bd8319d6eff59f1cdc6992`
- Title: fix: prefill rejection on Claude Opus 4.6+ (max-steps reminder + proxy paths)
- Verdict: **merge-after-nits**

## Review

Two related fixes bundled into one PR. The refactor in
`packages/opencode/src/provider/transform.ts:48-58` extracts the duplicated Claude-model
detection into `isClaudeModel()` and reuses it both in `normalizeMessages()`
(`transform.ts:69`) and in the caching gate in `message()` (`transform.ts:360`). That's
a clean DRY win — previously the empty-content filter only fired when the npm key was
exactly `@ai-sdk/anthropic`, which silently skipped the proxy / vertex-anthropic /
alibaba paths that Claude Opus 4.6+ now serves through, and was the proximate cause
of prefill rejections.

The one-line change in `packages/opencode/src/session/prompt.ts:1583` flips the
`MAX_STEPS` reminder message from `role: "assistant"` to `role: "user"`. This is
correct: Anthropic's API rejects an assistant turn with no tool calls when the
previous turn was already an assistant message (the "no consecutive assistant" rule),
so injecting the reminder as a user-role nudge is what unblocks the loop.

Nit: the test update at `packages/opencode/test/provider/transform.test.ts:1412` only
adds `id: "openai/gpt-4"` to the negative-case fixture but doesn't add a positive test
case asserting that `google-vertex-anthropic` / `@ai-sdk/alibaba` *do* now get filtered
— which is the actual regression being fixed. Worth adding before merge so the
expanded predicate has explicit test coverage. Otherwise this is a clear merge.
