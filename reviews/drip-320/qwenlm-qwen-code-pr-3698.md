# QwenLM/qwen-code PR #3698 — fix(acp): run auto compression before model sends

- Link: https://github.com/QwenLM/qwen-code/pull/3698
- SHA: `b280a3a808428b42c825aaa519217eced0c10750`
- Author: Jerry2003826
- Stats: +1054 / −41, 2 files

## Summary

Fixes #3652. The ACP `Session` was calling `GeminiChat.sendMessageStream` directly without running the existing automatic-compression flow first, so long ACP-driven conversations could blow past the model's context window. The fix invokes `tryCompressChat()` before each model send and re-reads the active chat afterward (because `tryCompressChat()` may replace the chat instance entirely). Coverage is added for normal prompts, swapped chat instances, tool-response follow-up sends, and slash commands that complete without a model send (which must skip compression).

## Specific references

- `packages/cli/src/acp-integration/session/Session.test.ts` L49–L62: new `expectCompressBeforeSend` helper using `mock.invocationCallOrder` to assert ordering between the compress and send mocks. This is the right tool for a "must happen before" invariant — far better than asserting call counts and hoping. Good test design.
- `packages/cli/src/acp-integration/session/Session.test.ts` L73–L101: `mockGeminiClient` is now extracted with `getChat` + `tryCompressChat` returning a `CompressionStatus.NOOP` result by default. Reasonable default — a test that wants to exercise the replace-chat path can override `getChat` after compression.
- `packages/cli/src/acp-integration/session/Session.test.ts` L131: adds `recordSlashCommand: vi.fn()` to the telemetry mock. Implies the production code now records slash commands; verify that telemetry call exists in `Session.ts` and isn't accidentally double-counted with an existing slash-command telemetry path.
- `packages/cli/src/acp-integration/session/Session.test.ts` L144–L148: replaces the inline `getGeminiClient` mock with the new `mockGeminiClient`. Also adds `getSessionTokenLimit: vi.fn().mockReturnValue(0)` — confirm the production code's compression check tolerates a `0` token limit (likely treated as "use default") rather than always-skip or always-trigger.
- `packages/cli/src/acp-integration/session/Session.test.ts` L358–L470: the four new sub-tests under `auto-compress` — normal prompt, swapped-chat, tool-response follow-up, and slash-command-no-send — directly mirror the four scenarios listed in the PR body. Coverage is symmetric with intent.
- `packages/cli/src/acp-integration/session/Session.ts`: not visible in the truncated diff window, but the contract that needs verifying is: every code path that currently calls `chat.sendMessageStream` (and `chat.sendMessage` if any) now goes through a `compressIfNeeded()`-style guard, AND re-reads `getGeminiClient().getChat()` after compression. Any path that misses the re-read will silently send on the stale chat and the bug returns.

## Verdict

verdict: merge-after-nits

## Reasoning

Fix is well-targeted, the ordering-via-`invocationCallOrder` pattern is the right way to test "compress must precede send", and the four-scenario test plan covers the realistic call sites. Pre-merge: confirm every `sendMessageStream`/`sendMessage` call site in `Session.ts` is gated, and confirm `getSessionTokenLimit() === 0` semantics match what the compression layer expects. Both are fast checks, not blockers.
