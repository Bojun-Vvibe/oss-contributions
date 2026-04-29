# sst/opencode #24898 — fix(session): remap compaction tail_start_id when forking

- **PR:** https://github.com/sst/opencode/pull/24898
- **Head SHA:** `a40622f62f4b07a203f6672a4aeffbf0bd4cccf7`
- **Author:** spark4862
- **Size:** +74 / −2 (2 files)
- **Closes:** #24700

## Summary
Surgical fix for `Session.fork` not remapping `CompactionPart.tail_start_id`. The forked session was inheriting an id from the parent that no longer exists in the fork's id space, so `filterCompacted` (`message-v2.ts`) walked past the intended retention boundary and pulled all pre-compaction history back into the prompt — observable as `context_length_exceeded` on the next request.

## Specific observations

1. **The fix at `packages/opencode/src/session/session.ts:619-628`** is exactly the minimal change: build a typed `p: MessageV2.Part`, then `if (p.type === "compaction" && p.tail_start_id) p.tail_start_id = idMap.get(p.tail_start_id)`. The `idMap.get` returning `undefined` on missing-tail cases (partial fork) is correct fallback behavior — it clears the stale ref instead of pointing at a nonexistent id, which is what `filterCompacted` already handles (no walk-target ⇒ no early stop bug).

2. **Test at `test/session/messages-pagination.test.ts:843-905`** exercises the right invariant: builds a multi-turn session with a real compaction (`addCompactionPart(session.id, c1, u2)`), asserts parent `filterCompacted` returns `[u2, a2, c1, s1, u3, a3]` (6 items), forks, asserts `childFiltered.length === parentFiltered.length`, then validates `tailPart.tail_start_id` is defined AND points to a message that actually exists in the child (`childFiltered.some((m) => m.info.id === tailPart.tail_start_id)`). That last assertion is the load-bearing one — it would have caught the bug.

3. **Regression provenance is documented** (commit `6f5a3d30f`, 2026-04-10, "keep recent turns during session compaction") which is the responsible-reviewer move and lets future maintainers grep the blame trail.

4. **One nit:** the new `const p: MessageV2.Part = {...part, ...}` constructs the part eagerly but only conditionally mutates `tail_start_id`; for stylistic consistency with the rest of the file (which favors object-spread builders), an inline ternary would be marginally cleaner — `tail_start_id: idMap.get(part.tail_start_id)` inside a conditional spread. The PR description's example (`...(part.type === "compaction" ...)`) is actually cleaner than what shipped. Cosmetic only.

## Verdict: `merge-as-is`

The fix is small, correct, well-tested with a discriminating assertion, and the failure mode (silent context-bloat) was high-severity user-visible. The cosmetic nit doesn't block — it's a one-line readability preference and the typed intermediate `p` arguably catches type drift better than a deeply-nested spread.

## Recommended actions
- Land as-is.
- Consider a follow-up sweep: are there other `MessageV2.Part`-discriminated fields besides `tail_start_id` that hold cross-message id references and would need the same `idMap` remap on fork? `parentID` is already covered; an audit of the `MessageV2.Part` union for any other `*_id` / `*ID` fields would prevent the next instance of this bug class.
