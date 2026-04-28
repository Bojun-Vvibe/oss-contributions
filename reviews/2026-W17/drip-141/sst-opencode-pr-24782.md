# sst/opencode#24782 — fix(session): remap compaction tail_start_id when forking

- **PR**: https://github.com/sst/opencode/pull/24782
- **Author**: @spark4862
- **Head SHA**: `27c3eb79350ab7ddc1b4e638e0311d0981983dc0`
- **Base**: `dev`
- **State**: CLOSED
- **Scope**: +71 / -0 across 2 files (1 src + 1 test)

## Summary

`Session.fork` deep-clones every part of a session into a new session, generating fresh message IDs and using an `idMap: Map<oldId, newId>` to rewrite cross-message references. The bug: `compaction` parts carry a `tail_start_id` field that points to the message ID where the compacted-tail begins, and the existing `updatePart` call at `packages/opencode/src/session/session.ts:606-612` was forwarding the original `tail_start_id` value into the cloned session — a stale pointer to a message ID that doesn't exist in the new session. Downstream, `MessageV2.filterCompacted` walks compactions and uses `tail_start_id` to keep messages from the tail; with a stale pointer, the child session's filter result diverges from the parent's. Fix is a 3-line conditional spread inserting the `idMap.get(part.tail_start_id)` translation only on `compaction` parts.

## Diff anchors

- `packages/opencode/src/session/session.ts:608-614` — the load-bearing change is the conditional spread `...(part.type === "compaction" && part.tail_start_id ? { tail_start_id: idMap.get(part.tail_start_id) } : {})`. Two correctness points worth naming:
  - The discriminator `part.type === "compaction"` is required because `tail_start_id` is only on the compaction shape — adding it to non-compaction parts would either type-error or silently widen the persisted shape.
  - The `&& part.tail_start_id` truthy guard preserves the existing semantics for compaction parts that don't have a tail_start (which is a valid state — compactions early in a session's life can lack a tail).
- `packages/opencode/test/session/messages-pagination.test.ts:843-906` — new `test("fork remaps compaction tail_start_id so filterCompacted matches parent")` is correctly shaped:
  - Constructs a parent session with two pre-compaction turns (`u1/a1`, `u2/a2`), then a compaction message `c1` with `addCompactionPart(session.id, c1, u2)` (so `tail_start_id == u2`), a summary assistant `s1`, and one post-compaction turn (`u3/a3`).
  - Pins the parent invariant first at `:889`: `parentFiltered.map(...id) === [u2, a2, c1, s1, u3, a3]` — this proves the test's compaction-walking logic is sane before it asks the child to match.
  - Forks, runs `filterCompacted` on the child, and asserts `childFiltered.length === parentFiltered.length` plus the load-bearing pointer-validity assertion at `:902`: `childFiltered.some((m) => m.info.id === tailPart.tail_start_id) === true`. This is the right shape — it doesn't pin the new ID's specific value (which would couple the test to ULID generation), it pins the structural invariant that the rewritten pointer is dereferenceable inside the new session.
- `packages/opencode/test/session/messages-pagination.test.ts:32-34` — adds `fork(input)` to the local `svc` test harness. Mechanical, but worth checking that the harness is the only fork callsite the test uses (it is).

## What I'd push back on (nits)

1. **No coverage for the negative cell** — a compaction part with `tail_start_id` pointing to a message that isn't in the parent session (defensive: should the spread silently emit `undefined`, or should it throw?). With the current code, `idMap.get(missing)` returns `undefined` and the cloned part has `tail_start_id: undefined`, which is the same shape as "no tail." That's probably the right defensive choice but it deserves a one-line comment at `:610` so future readers don't mistake it for a bug.
2. **PR is CLOSED, not MERGED** — the substance of the fix is correct, but I'd want to know from the author or maintainer whether this was closed as superseded by another PR (search "tail_start_id" in recent commits before re-implementing). If superseded, this review is for the historical record; if abandoned, the fix is small enough to land directly.
3. **`addCompactionPart` helper isn't shown in the diff** — I'm trusting that the existing test helper at the top of `messages-pagination.test.ts` correctly constructs a compaction part with a `tail_start_id` field. Reviewer should verify by reading the helper definition.

## Verdict

**merge-as-is** (modulo confirming why the PR is in CLOSED state). The fix is minimal, the discriminator + truthy-guard pattern is correct, and the test pins both the structural correctness (length match) and the pointer-validity invariant (`childFiltered.some(... === tailPart.tail_start_id)`) without coupling to ID generation specifics.

Repo coverage: sst/opencode (session/fork semantics).
