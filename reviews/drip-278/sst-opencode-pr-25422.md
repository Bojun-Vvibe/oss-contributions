# Review — sst/opencode#25422

- PR: https://github.com/sst/opencode/pull/25422
- Title: fix(acp): drain message events before returning end_turn
- Head SHA: `edae14d0eb6655d45bef212e7d4b48439b5188ad`
- Size: +53 / −0 across 1 file (`packages/opencode/src/acp/agent.ts`)
- Verdict: **merge-after-nits**

## Summary

Fixes an ACP protocol violation where `prompt()` returned
`stopReason: "end_turn"` while trailing `message.part.delta` events for
the same turn were still queued in the SDK event stream. The result
was `agent_message_chunk` frames arriving on the wire AFTER the RPC
reply, which clients legitimately treat as a bug ("text appearing
post-end_turn").

The fix introduces a per-message-id resolver map and gates the prompt
return on observing `message.updated` with `time.completed !==
undefined` for the assistant reply.

## Evidence

- `packages/opencode/src/acp/agent.ts:150-151` — two new collections
  on `Agent`:
  - `messageCompletionResolvers: Map<string, () => void>`
  - `completedAssistantMessageIds: Set<string>`
- `packages/opencode/src/acp/agent.ts:275-285` — new `message.updated`
  case in the event subscription that records the completed id and
  drains any pending resolver. Critically this lives inside
  `runEventSubscription` which processes events sequentially via
  `for await` and awaits each `handleEvent`, so observing the
  completed event guarantees every prior `message.part.delta` has
  already been forwarded to ACP. The PR author calls this out in the
  block comment above `waitForMessageCompletion`.
- `packages/opencode/src/acp/agent.ts:549-577` — new
  `waitForMessageCompletion(messageId, timeoutMs)` returns
  immediately if the id is already in the completed set, otherwise
  registers a resolver that is also fired by a 5000ms `setTimeout`
  fallback.
- `packages/opencode/src/acp/agent.ts:~1527` — the actual call site
  in the prompt path: `if (msg?.id) { await
  this.waitForMessageCompletion(msg.id, 5000) }` placed before
  `sendUsageUpdate`.

## Notes / nits

- `completedAssistantMessageIds` grows unboundedly across the lifetime
  of an `Agent`. For a long-lived ACP session this is a slow memory
  leak (one string id per assistant turn). Consider either pruning the
  entry inside `waitForMessageCompletion` after it resolves, or using
  a bounded LRU. Same goes for `messageCompletionResolvers` if a
  caller never awaits — the timer cleans the map but only after 5s.
- The 5000ms timeout silently resolves on stall, so a genuinely stuck
  stream will still return `end_turn` with chunks possibly trailing
  after — the original bug, just with a 5s ceiling. A `log.warn` on
  the timer fallback path would make this much easier to diagnose in
  the field.
- `setTimeout` is not cleared when the resolver fires from
  `message.updated`. The timer itself is harmless (the closure noops
  on the second call thanks to `settled`), but Node will keep the
  process alive an extra ≤5s during shutdown. Wrapping in
  `clearTimeout` on success would tidy this up.

Ship after at least the unbounded-set nit is addressed; the others are
follow-ups.
