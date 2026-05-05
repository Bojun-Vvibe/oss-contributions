# sst/opencode #25925 — fix(provider): generate fallback ID for tool calls missing 'id' in streaming

- URL: https://github.com/sst/opencode/pull/25925
- Head SHA: `40178e0342ab3dd48ef82d5dea102d9cf8af68d4`
- Diff: +160 / -4 across 2 files (1 src under `provider/sdk/vsc-redacted-acp/chat/openai-compatible-chat-language-model.ts`, 1 new test)

## Findings

1. The semantic change is at `openai-compatible-chat-language-model.ts:541-545` (per the diff hunk starting at line 539): the streaming branch previously threw `InvalidResponseDataError` when `toolCallDelta.id == null`; now it assigns `toolCallDelta.id = generateId()` and emits a `console.warn`. This brings the streaming path in line with the non-streaming path, which the PR description correctly cites as already using `generateId()` at lines 247/591/634/668. The asymmetry was a real bug — providers like NVIDIA's `moonshotai/kimi-k2.5` only omit `id` on streaming tool deltas, so the non-stream parity argument is the right framing for accepting the change.
2. `generateId` is already imported from `@ai-sdk/provider-utils` (line 17 per author note), so no new dependency footprint.
3. Test coverage at `test/provider/.../openai-compatible-tool-call-id-fallback.test.ts:1-158` is solid — three fixtures (missing id, real id preserved, multiple missing ids), each verifying the emitted `tool-input-start` / `tool-call` events have non-empty IDs and `finish_reason: tool-calls` is propagated. The "preserved real id" case is the critical regression guard, since it confirms the fallback only fires when the field is actually `null`.
4. Minor: `console.warn` is unguarded and will fire on every malformed chunk. For providers that consistently omit `id` (e.g., NVIDIA), this can spam stderr in long sessions. Consider gating behind the existing debug logger, or warn-once per provider instance via a `Set<string>` on the model. Not a blocker.
5. Behavioural note: the fallback ID is generated independently per chunk in the no-`id` case. The accumulator branch `if (toolCalls[index] == null)` only enters once per `index`, so the same generated ID will be reused for subsequent argument deltas at the same index — that's correct (matches OpenAI semantics), but a one-line comment clarifying this would help future readers.

## Verdict

**Verdict:** merge-after-nits
