# sst/opencode#25255 — fix(processor): fix doom loop detection scope and filter order

- **PR**: https://github.com/sst/opencode/pull/25255
- **Author**: qz1543706741
- **Head SHA**: `5ea7af24e16613e5871b424d3f4b20ecec4355f4`
- **Files**: `packages/opencode/src/session/processor.ts`
- **Verdict**: **merge-after-nits**

## Context

The doom-loop detector at `processor.ts:302-322` (pre-fix) sliced the **last N parts of the current assistant message** then required *all* of them to be the same tool with identical arguments. Two structural bugs:

1. **Scope too narrow.** Slicing `MessageV2.parts(ctx.assistantMessage.id)` only sees the *current* assistant message. A real doom loop typically spans across `assistant → tool-result → assistant → tool-result …` boundaries, so by the time the assistant emits the Nth identical call it's been across multiple message rounds and the slice misses prior identical calls entirely.
2. **Filter-order inverted.** "Take the last N then check all match" rejects the loop the moment a single non-matching tool call lands in the last-N window, even if there are still N identical calls intermingled.

## What's right

- **Scope expansion to "since last user turn".** New `recentMatchingToolParts` at `:155-171` uses `MessageV2.filterCompactedEffect(ctx.sessionID)` then `findLastIndex(role === "user")` and slices everything after that boundary. This is the correct semantic window — a doom loop is "the assistant repeating itself within a single user turn", and the new predicate matches that definition exactly.
- **Filter-then-take order.** `:165-170` filters first (`type === "tool"` && same `tool` && `state.status !== "pending"` && `JSON.stringify(state.input) === JSON.stringify(args)`) then `.slice(-DOOM_LOOP_THRESHOLD)`. This means N intermingled identical calls trigger detection regardless of what other tools fire in between — the load-bearing inversion vs. the prior take-then-all-match.
- **Call-site collapse.** `:323-325` shrinks from a 14-line take-then-all-match block to a 2-line `recentParts = yield* recentMatchingToolParts({tool, args}); if (length !== threshold) return`. The check is now expressible as a single length comparison because the predicate is fully encapsulated in the helper.
- **Pending-state exclusion preserved.** The `state.status !== "pending"` filter at `:166` correctly excludes the in-flight current call from the loop count, so detection fires on the *Nth* completed identical call rather than the (N-1)th completed plus the in-flight one (which would be off-by-one).

## Risks / nits

- **`JSON.stringify` is order-sensitive on object keys.** Two semantically identical tool calls with different key insertion order (e.g. `{path: "a", recursive: true}` vs `{recursive: true, path: "a"}`) would not match. JS engines preserve insertion order so the model would have to genuinely emit different orders, but consider canonicalizing via sorted-key replacer or `JSON.stringify(value, Object.keys(value).sort())` to harden against future model behavior changes.
- **`filterCompactedEffect` walks the whole session every check.** For long sessions with no compaction, this is O(messages × parts) per tool-call boundary. Probably fine in practice (compaction kicks in) but worth a comment noting the assumption that compaction bounds the walk.
- **No test coverage in the diff.** The behavior change is subtle (detection now spans message boundaries and ignores intermingled non-matches) — a regression test asserting "5 identical calls with intermingled `read` calls trigger detection" and "5 calls split across 2 user turns do NOT trigger" would pin the new semantics.

## Verdict

**merge-after-nits.** Two real structural bugs (scope-too-narrow + filter-order-inverted) closed at one helper. The semantic window — "assistant repeating itself within a single user turn" — is the right definition for "doom loop" and the helper makes it the source-of-truth.
