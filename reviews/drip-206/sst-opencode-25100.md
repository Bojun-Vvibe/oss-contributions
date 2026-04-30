# sst/opencode #25100 — feat(opencode): cache-aligned compaction to reuse prefix cache

- Head SHA: `3a48d1b4f162f09243e458fc3e3e465e4719cab4`
- Files: `packages/opencode/src/session/compaction.ts`, `packages/opencode/src/session/prompt.ts`
- Size: +177 / -75

## What it does

Today the compaction request is built as a synthetic LLM call with `system: []`,
`tools: {}`, and a `hidden`-filtered message slice. Because that prefix differs
from the live agent loop's prefix byte-for-byte, the provider's prompt cache
never matches and the entire history is re-billed at full input price. This PR
threads an optional `ResolvedContext { agent, system[], tools, user }` through
`Interface.compact(...)` (compaction.ts:204, :358) so the caller can hand the
exact same `system` strings and `tools` map that the agent loop just used; when
`resolved` is supplied, compaction skips its own `agents.get("compaction")`
override (compaction.ts:392-395), drops the `hidden` index filter
(compaction.ts:397-401), skips `stripMedia`/`toolOutputMaxChars` so the message
serialization matches the live path (compaction.ts:417-421), and dispatches
through `processor.process(...)` with `toolChoice: "none"` plus the resolved
agent/system/tools (compaction.ts:455-475).

The author also makes `processor` optional on the prompt builder
(prompt.ts:364) so the live loop can call it once with `processor: undefined`
purely to materialize the cache-aligned tool definitions, with a clarifying
comment at prompt.ts:372 and a `messageID` fallback at prompt.ts:374. The
`metadata` and `completeToolCall` callbacks become no-ops when no processor is
present (prompt.ts:386-407).

## What works

The split between the legacy and `resolved` branches is conservative — every
existing call site keeps the old `userMessage` / empty-tools / hidden-filter
behavior because `input.resolved` is `undefined`. The `toolChoice: "none"`
override at compaction.ts:472 is correct: cache-aligned does not mean we want
the summarizer to actually emit tool calls. Threading `parentSessionID` and
`permission` from the live `Session` (compaction.ts:455, :461-462) keeps
sub-session attribution intact in the rolled-up summary.

## Concerns

1. **Cache key fragility.** The 90% saving claim depends on the resolved
   `system[]` and `tools` map being byte-identical to the loop's last request.
   The PR adds no test asserting that round-trip — a future change to
   `system` ordering, tool schema serialization, or even `user.text` whitespace
   would silently regress to the old cost without any signal.
2. **`hidden` filter dropped wholesale.** When `resolved` is set the previously
   `hidden` (already-summarized) messages are now re-fed to the summarizer.
   That is necessary for the cache match, but it changes the *semantics* of
   the summarizer's input: it now sees prior summary turns inline. Worth a
   note in the buildPrompt template that the summarizer must collapse
   already-collapsed material.
3. The diff strips the `stripMedia: true, toolOutputMaxChars: TOOL_OUTPUT_MAX_CHARS`
   safety net (compaction.ts:417-421). For very large image-bearing histories
   this could push compaction itself over the model's input limit, which is
   exactly when compaction is most needed. Consider keeping the cap but
   applying it only to messages newer than the cache hit-rate target.

Verdict: merge-after-nits
