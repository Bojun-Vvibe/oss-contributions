# sst/opencode #24170 — preserve assistant tool_calls in openai-compatible replay

**Link:** https://github.com/sst/opencode/pull/24170
**Tag:** schema-drift, silent-default

## What it does

Closes #24090. On replay, when a `tool` result message is present but the
preceding assistant message has lost its matching `tool-call` parts, the
provider serializer emits an unpaired `tool_call_id` and OpenAI-compatible
backends reject the turn. The fix walks the rebuilt `ModelMessage[]` from
`MessageV2.toModelMessagesEffect` and, for every `tool` message, scans
backwards to the nearest assistant entry and synthesizes any missing
`tool-call` parts from a `replayToolCalls` map keyed on `callID` (recovered
from the persisted `WithParts` "tool" entries, including original
`toolName`/`input`). The same repair pass is also wrapped as a send-time
guard in `session/llm.ts` and applied **twice** around the
`ProviderTransform.message` call — once on the input prompt, once on the
already-transformed output — gated on
`input.model.api.npm === "@ai-sdk/openai-compatible"` and
`hasToolCalls(messages)`.

## What it gets right

- Repair is **idempotent** and **bounded**: `existingToolCallIDs` filter
  stops it from duplicating already-present `tool-call` parts, and the
  flatMap over a missing-only set keeps allocations linear.
- Provider scope is narrow: only the OpenAI-compatible NPM api gets the
  send-time guard, so OpenAI/Anthropic native paths stay untouched.
- New regression coverage in both `message-v2.test.ts` (replay rebuild) and
  `llm.test.ts` (send-time guard) catches the two layers independently.
- Preserves `providerExecuted` and `providerOptions` flags when rehydrating,
  not just shape.

## Concerns / risks

- **Double-application** in the `transformParams` hook (repair → transform →
  repair again) hides the actual contract violation: either
  `ProviderTransform.message` is dropping `tool-call` parts (a bug worth
  fixing at the source) or the input was already broken. The second call is
  a band-aid that will silently keep working even after the underlying
  transform is fixed, masking the regression.
- The rebuild scans backwards for "nearest preceding assistant"; if the
  history was corrupted such that two unrelated `tool` blocks reference the
  same assistant, the synthesized calls land on the wrong assistant.
- `differentModel ? undefined : providerMeta(part.metadata)` silently
  discards provider metadata on cross-model resume — fine for replay
  fidelity, but worth a comment explaining why.

## Suggestion

Add a debug-log when the send-time guard actually fires (count of
synthesized calls, model id) so operators can see when
`ProviderTransform.message` is dropping parts, then schedule a follow-up to
delete the second-pass repair once that root cause is fixed. Otherwise the
guard becomes load-bearing infra nobody owns.
