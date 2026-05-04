# sst/opencode#25683 — fix(acp): drain message events before returning end_turn

- **Head SHA:** `edae14d0eb6655d45bef212e7d4b48439b5188ad`
- **Author:** community
- **Size:** +53 / −0, 1 file (`packages/opencode/src/acp/agent.ts`)
- **Closes:** #25421

## Summary

`Agent.prompt()` was returning `stopReason: "end_turn"` as soon as `sdk.session.prompt()` resolved, but trailing `message.part.delta` events were still queued in the SDK event stream. Those deltas were forwarded as `agent_message_chunk` after the RPC reply landed — a wire-level ACP protocol violation where text appeared after `end_turn`. The fix tracks per-message completion via `message.updated` and awaits it (with a 5s timeout) before returning.

## Specific citations

- `agent.ts:150-151` — adds `messageCompletionResolvers: Map<string, () => void>` and `completedAssistantMessageIds: Set<string>`. State on the Agent instance, no per-call cleanup besides the `delete()` calls in resolver/finish — see risk below.
- `agent.ts:275-285` — new `message.updated` case that, when `info.role === "assistant"` and `info.time.completed !== undefined`, marks the id complete and fires any pending resolver. Correct event to gate on; matches SDK semantics.
- `agent.ts:549-575` — `waitForMessageCompletion(messageId, timeoutMs)` returns immediately if already completed, else races a one-shot resolver against `setTimeout(finish, timeoutMs)`. The internal `settled` guard prevents double-resolve.
- `agent.ts:1527-1530` and `agent.ts:1556-1559` — two prompt-return sites both call `await this.waitForMessageCompletion(msg.id, 5000)` before `sendUsageUpdate` / returning. Good, both branches are covered.

## Verdict: `merge-after-nits`

The core mechanism is sound and the comment in `agent.ts:546-558` is unusually clear about *why* the drain is necessary (sequential `for await` in `runEventSubscription`).

Nits worth addressing before merge:

1. **Unbounded growth of `completedAssistantMessageIds`.** The set is added to on every assistant `message.updated` with `time.completed` (`agent.ts:278`) and never cleared. Long-lived agent processes will accumulate one entry per turn forever. Either prune after `waitForMessageCompletion` consumes it, or bound it (e.g., LRU/recent ids only) since once `prompt()` has drained for a given `msg.id` the entry is dead weight.
2. **5000 ms timeout is silent on miss.** If the SDK never emits a completion event (regression, server bug), `prompt()` returns successfully with `end_turn` after a 5s stall and zero log signal. A `log.warn("waitForMessageCompletion timeout", { messageId })` inside `finish()` when triggered by the timer would make this debuggable in the field.
3. **No test.** The PR description references manual repro but no unit/integration test guards regression. A fake event stream that emits `message.part.delta` then `message.updated` could assert ordering against a recording `connection`.

None of these block correctness; they're hardening. Mechanism itself is the right fix.
