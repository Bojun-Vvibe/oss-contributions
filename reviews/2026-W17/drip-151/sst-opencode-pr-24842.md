# sst/opencode#24842 — fix(session): cache messages across prompt loop to preserve prompt cache byte-identity

- **Repo:** [sst/opencode](https://github.com/sst/opencode)
- **PR:** [#24842](https://github.com/sst/opencode/pull/24842)
- **Head SHA:** `d5baeca8576f6510f35323be3c9d09880d6dfd3a`
- **Size:** +28 / -1 (one file: `packages/opencode/src/session/prompt.ts`)
- **State:** OPEN
- **Closes:** #24841

## Context

Real cost incident, not a hypothetical. Author shows real-session telemetry
(Opus 4.7, 1M context, April 21st): cache writes at $2,264/day vs cache reads
at $1,234/day, with 95% of "rapid cache busts" (<60s gap) preceded by a
tool-bearing message. The diagnosis: between tool-call iterations of the
prompt loop, the same message gets re-serialized differently because tool
parts transition `pending → completed`, and Anthropic's prompt cache matches
on **byte-identity of the message prefix**, so the changed bytes invalidate
everything from that position forward.

## Design analysis

The fix at `packages/opencode/src/session/prompt.ts:1279-1295` introduces a
`msgs` cache and a `needsFullReload` flag scoped to the loop:

```ts
let msgs: MessageV2.WithParts[] | undefined
let needsFullReload = true

while (true) {
  // ...
  if (needsFullReload || !msgs) {
    msgs = yield* MessageV2.filterCompactedEffect(sessionID)
    needsFullReload = false
  }
```

The clever bit is the asymmetry between the four "structural change" exits
and the tool-call continuation exit. Subtask handling
(`prompt.ts:1346-1348`), overflow recovery (`:1359-1361`), and explicit
compaction (`:1370-1372`) all set `needsFullReload = true` because those
paths actually rewrite the conversation shape. The tool-call continuation
arm at `:1505-1518` instead does an **append-only merge** keyed on
`msg.info.id`:

```ts
const fresh = yield* MessageV2.filterCompactedEffect(sessionID)
const existingIds = new Set(msgs!.map((m) => m.info.id))
for (const msg of fresh) {
  if (!existingIds.has(msg.info.id)) {
    msgs!.push(msg)
  }
}
```

This is the load-bearing trick: the only message whose tool parts transition
`pending → completed` between API calls is the most recent assistant
message. By **not re-reading it from the DB**, its serialized form stays
byte-identical with what Anthropic cached on the prior turn. All earlier
messages were already `completed` when first sent, so their bytes are
stable.

## Risks / nits

1. **In-memory mutation drift.** The cached `msgs` array is now mutated in
   place across iterations (`msgs!.push(msg)`). If anything inside the loop
   mutates a message object (not the array — an individual `msg.info` or
   `msg.parts`), that mutation now persists across iterations instead of
   being reset by the next `filterCompactedEffect` reload. Worth a one-line
   comment that consumers must treat `msgs` entries as immutable.

2. **Subtask path dependency.** `handleSubtask({...msgs})` at line 1347
   receives the cached array *before* `needsFullReload = true` fires. If
   `handleSubtask` ever mutates the array (e.g., splice), the next reload
   recovers, but a non-subtask code path that took a reference to the same
   array would observe the mutation. Defensive copy at the subtask boundary
   would be cheap insurance.

3. **`filterCompactedEffect` semantics on append-only path.** The
   continuation arm calls `filterCompactedEffect(sessionID)` again
   (`:1512`), which is the same effect that does the full reload. If
   compaction silently fired between iterations and `filterCompactedEffect`
   now returns a *truncated* tail, the `existingIds` filter would skip
   re-adding the truncated head — but those messages are still in `msgs`
   from the prior iteration, so the cached array would now contain
   compacted-away messages. The compaction explicit-trigger arm at `:1370`
   handles this, but a *background* compaction would not. Worth an
   assertion or an explicit "compaction must go through `needsFullReload`"
   invariant comment.

4. **Diagnostic gap.** The fix doesn't add a counter or log line for "skipped
   full reload" vs "did full reload". Given the cost magnitude in the PR
   body, a `slog.info("loop", { step, fullReload: <bool>, msgCount: ... })`
   adjustment at `:1283` would make the cache-hit-rate observable in
   production logs.

## Verdict

**merge-after-nits.** The diagnosis is correct and well-evidenced (real
telemetry, cost numbers, hit-rate analysis). The fix is minimal (28 lines)
and the four "needs full reload" sites look exhaustive against the loop's
structural-change branches. Nits 2 and 4 are pre-merge; 1 and 3 are
follow-up comments / invariant docs.

## What I learned

- Prompt-cache invalidation in vendor caches that match on **byte-identity
  of the prefix** is brutal: a single byte change in any earlier message
  invalidates the whole prefix from that point. The fix is "don't re-read
  what you've already sent."
- The `pending → completed` tool-part state transition is a quiet source of
  byte drift across loop iterations because the serialization function
  branches on state. Caching the serialized form (vs the source-of-truth
  state) is the right answer at this seam.
- Append-only merge keyed on stable IDs is a clean way to make "incremental
  reload" safe without a full diff/patch protocol, as long as the
  identity-preserving invariant (only the tail message changes) holds. The
  compaction/subtask/overflow exits rightly bypass the optimization
  because *those* paths break the invariant.
