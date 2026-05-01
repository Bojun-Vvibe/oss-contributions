# sst/opencode#25195 — fix(opencode): synthesize reasoning-start for incomplete streams

- Link: https://github.com/sst/opencode/pull/25195
- Head SHA: `d38241026addc69e698e0e09dfe4c20913aa4ff0`
- Files: `packages/opencode/src/session/llm.ts` (+18/−0)

## Notes
- Surgical 18-line `wrapStream` middleware insertion at `session/llm.ts:400-416` that intercepts AI-SDK reasoning chunks and synthesizes a missing `reasoning-start` event before any orphan `reasoning-delta`/`reasoning-end` arrives — fixing providers (e.g. anthropic-via-bedrock failure modes, mid-stream resumes after compaction) that emit deltas without a corresponding start, which would otherwise crash downstream reasoning-block accumulators that key on `started.has(chunk.id)`.
- Per-`id` `Set` tracker at `:402` (`const started = new Set<string>()`) is the right scope: state is per-stream invocation, not global, so concurrent streams can't poison each other's id-tracking. The synthesis path at `:407-411` adds the id to `started` *before* enqueuing the synthetic start, preventing a duplicate-synthesize race if the same id appears in both a stray delta and a stray end.
- Pass-through of the original chunk at `:413` after the synthetic enqueue preserves the original `reasoning-delta`/`reasoning-end` payload, so downstream consumers see `[start, delta, end]` instead of `[delta, end]` — which is the canonical AI-SDK contract every consumer assumes.
- `pipeThrough(fix)` at `:415` cleanly composes with the existing transform stack, no changes to `result` shape.

## Nits
- Zero unit test for the load-bearing synthesis arm — adding a fixture stream emitting `[reasoning-delta{id:"r1"}, reasoning-end{id:"r1"}]` and asserting the consumer sees `[reasoning-start{id:"r1"}, reasoning-delta, reasoning-end]` would pin the contract so a future "let's filter empty deltas" refactor surfaces immediately.
- `chunk.id` is read without a guard on chunks that lack `id` (some provider variants emit `reasoning-delta` with the id on a sibling field). A `if (chunk.id == null) { controller.enqueue(chunk); return }` short-circuit at `:404` would defend against the next provider-quirk discovery.

**Verdict: merge-after-nits**
