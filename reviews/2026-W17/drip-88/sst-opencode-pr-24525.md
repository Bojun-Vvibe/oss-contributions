---
pr: 24525
repo: sst/opencode
sha: 1dbfda561408a4eb8944b93145de69093a218879
verdict: merge-after-nits
date: 2026-04-27
---

# sst/opencode #24525 — Mitigate memory spikes from event backlog and shell output

- **Author**: zjy-dev
- **Head SHA**: 1dbfda561408a4eb8944b93145de69093a218879
- **Size**: +183/-49 across `bus/index.ts`, `server/routes/global.ts`, `server/routes/instance/event.ts`, `session/prompt.ts`, `util/queue.ts`.

## Scope

Three independent memory-amplification paths were producing multi-GB RSS and OOM-kills in long sessions; this PR lands one fix per path:

1. **Wildcard pubsub backlog**: `Bus` swaps `PubSub.unbounded<Payload>()` → `PubSub.sliding<Payload>(1024)` so a stalled wildcard subscriber (SSE stream, plugin) drops oldest events instead of retaining unbounded history.
2. **SSE per-connection queue overflow**: both `GlobalRoutes` and `EventRoutes` SSE endpoints now bound their `AsyncQueue` to 256 entries and explicitly stop the connection on overflow rather than buffering forever.
3. **Streaming tool-metadata flood**: `metadata()` calls during tool execution are throttled at 100ms / deduplicated via a JSON fingerprint, and shell-tool output spills to disk after `limits.maxBytes` while keeping only a sliding 30 KB preview in memory and a sliding chunk window for the final output.

## Specific findings

- `packages/opencode/src/bus/index.ts:48-52` — `PubSub.sliding<Payload>(1024)` is the right primitive. Sliding (vs `dropping`) keeps the *most recent* events, which is what you want for a debugging/observability bus where stale events are less interesting than current ones. 1024 is a reasonable default; consider exposing as `OPENCODE_BUS_WILDCARD_CAPACITY` env override for diagnosis.
- `packages/opencode/src/util/queue.ts:1-32` — `AsyncQueue` gains a `maxSize` constructor arg (default `Number.POSITIVE_INFINITY` preserves prior behavior) and a `size` getter. `push()` now drops oldest when over `maxSize`. **One subtle bug**: when a `resolve` is waiting (`this.resolvers.shift()` returns truthy), the item is delivered directly and never bounds-checked. That's fine behaviorally (a waiting consumer means there's nothing buffered) but the `while (this.queue.length > this.maxSize)` only runs in the else-branch, so document the invariant ("bounded queue only applies when consumer is slower than producer").
- `packages/opencode/src/server/routes/global.ts:25-44` and `routes/instance/event.ts:38-67` — the `(push, stop)` callback shape replaces `(q)`-as-queue. Good encapsulation: subscribers no longer have direct queue access, which means they can't accidentally `q.push(null)` or read `q.size` from outside. The overflow-then-stop policy at `:32-39` is correct: dropping individual SSE messages would silently desync the client; closing the connection forces the client to reconnect from a known state.
- `packages/opencode/src/server/routes/global.ts:75-77` — the heartbeat interval now flows through the bounded `push`, so a stalled HTTP socket triggering 25 heartbeats × 256-cap will close the stream within ~250 seconds. Reasonable failsafe.
- `packages/opencode/src/session/prompt.ts:384-418` — the `toolMetadata` map keyed by `toolCallId` plus the 100ms / fingerprint dedup is the right shape. **The fingerprint via `JSON.stringify(input)` will mis-dedup on key-order differences** (e.g., `{title:"a",metadata:{x:1,y:2}}` vs `{metadata:{y:2,x:1},title:"a"}`). For the call sites this is probably fine (tool implementations construct objects deterministically), but if a future tool uses `Object.assign` or spread from variable sources, the dedup will silently fail. Consider `JSON.stringify` over a sorted key list, or a structural-equality helper.
- `packages/opencode/src/session/prompt.ts:401-414` — the `metadata` callback now preserves prior `title`/`metadata`/`time.start` from the running state when the new value is `undefined` (`val.title ?? running?.title`, `time: { start: running?.time.start ?? current }`). This is a real correctness improvement: previously a partial `metadata({title: "x"})` would clobber the existing metadata to `undefined`. Worth a brief test.
- `packages/opencode/src/session/prompt.ts:866-940` — the shell-output spill-to-disk path: in-memory chunks bounded at `limits.maxBytes * 2`, transition to `createWriteStream(outputPath, { flags: "a" })` when the in-memory buffer exceeds `limits.maxBytes`. Good. **Resource leak risk at `:935-942`**: `outputSink.end()` is wrapped in a Promise that resolves on either `'end'` callback or `'error'` event — but `end()`'s callback fires on flush completion, and if `'error'` fires *first* (e.g., disk full), the callback never fires and the Promise resolves via the `error` listener; you do NOT call `stream.destroy()`. Fine for the normal path but consider explicit `stream.destroy()` on error to release the FD immediately.
- `packages/opencode/src/session/prompt.ts:464` and `:543` — the `toolMetadata.delete(options.toolCallId)` cleanup wrapped in `Effect.ensuring(...)` is the right shape: runs whether the tool succeeds, fails, or is interrupted. Good defensive cleanup.

## Risk

Medium. Three independent improvements landing in one PR is reviewable but ideally would be three commits/PRs (the bus change, the SSE bounding, and the shell spill are orthogonal). The shell spill-to-disk path is the highest-risk change — it adds disk I/O to the hot streaming path, mutates state across `Effect.gen` boundaries, and the cleanup-on-error story for `outputSink` is not bullet-proof. Tests should cover at least: (a) shell output exceeding `maxBytes` writes to disk, (b) abort mid-stream cleans up the sink, (c) SSE queue overflow closes the connection, (d) bus wildcard at capacity drops oldest. None of those appear in the diff.

## Verdict

**merge-after-nits** — solid diagnoses, well-scoped fixes. Asks: (1) at least one test per memory-amplification path, (2) replace `JSON.stringify(input)` fingerprint with a key-order-stable variant or document the assumption, (3) explicit `stream.destroy()` on error in the shell-sink cleanup, (4) consider splitting into 3 PRs on the next iteration so each can be reverted independently if a regression surfaces.
