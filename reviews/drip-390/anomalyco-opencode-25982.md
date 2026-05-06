# Review: anomalyco/opencode#25982 — feat: Preserve todo progress during compaction

- Head SHA: `3aa36c5a82048cc526e17cf2fffbe66f8b590531`
- Files: 4 (+329 / -11)
- Author: pzy2000
- Verdict: **merge-after-nits**

## Summary

Preserves `TodoWrite` state and active plan path across summarization compaction by writing deterministic anchor blocks (`<!-- opencode-compaction-anchors:start -->` / `:end -->`) into the compacted summary text, then stripping and rewriting them on every subsequent compaction so stale todo statuses don't accumulate. Closes #18071, #18564, #5934, #15096.

## Specific evidence

- **Anchor sentinels** at `packages/opencode/src/session/compaction.ts:45-46` give a unique, comment-shaped delimiter pair — comment shape is correct because the surrounding template at `:46-103` is Markdown that the model is told to render verbatim, so HTML comments survive without polluting rendered output.
- **`stripAnchors()`** at `compaction.ts:115-123` walks the summary in a `while (true)` loop scanning for new `ANCHOR_START` after each successful slice, so multiple stacked anchor blocks (which can occur if the model copies a previous summary verbatim) are all removed. Unmatched start with no end is also handled (`end === -1` slice-and-trim at `:120`) — defensive against a partially-truncated summary.
- **`formatAnchors()`** at `:131-152` only emits the wrapper when at least one section is present (`if (!sections.length) return undefined` at `:151`), so empty-todos + no-plan correctly produces no anchor block at all, not an empty `<!-- start --><!-- end -->` ghost.
- **`cleanAnchorLine()`** at `:111-113` collapses whitespace runs and falls back to `"(empty)"` for empty content. This matters because `Todo.Info.content` is user-supplied free text — multiline content would otherwise break the single-line `- status / priority: content` row format at `:140`.
- **`persistAnchors()`** at `:362-398` finds the *last* text part of the compacted assistant message (`textParts.at(-1)` at `:373`) and only mutates that one, while still calling `stripAnchors()` on every other text part at `:387` — this scrubs any leaked anchor block from prior parts (e.g. if the model echoed the anchors mid-summary).
- **Idempotent updatePart** at `:391-394`: skips `session.updatePart(part)` when text didn't change, so repeated compactions on already-correct summaries don't churn the storage layer or fire spurious `EventV2` updates.
- **Plan detection** uses `fs.existsSafe(plan)` at `:355` with `Effect.catch(() => Effect.succeed(false))` — correct, since `existsSafe` shouldn't throw but a missing layer dependency in tests would otherwise crash compaction. Worth confirming `existsSafe` is the right primitive vs `Effect.tryPromise`.
- **Tests** at `test/session/compaction.test.ts:1-218` (+218): three regression scenarios (preserve current todos / replace stale anchors / preserve plan path) — covers the headline contract.

## Nits

1. `appendAnchors()` at `:125-129` doesn't handle the case where `input.summary` already ends with a trailing anchor block plus trailing whitespace — `stripAnchors()` does trim but the join-on-`"\n\n"` will produce a single empty line gap, fine but worth a comment.
2. The new layer dependency on `Todo.Service` and `AppFileSystem.Service` at `:270-271, 282-283` is correct but no `Layer.provide` shim is documented for downstream embedders that wire `compaction.layer` themselves — a CHANGELOG note on the new requirement would prevent silent build breaks.
3. `TodoWrite`-derived rows expose status/priority/content in plain Markdown — fine, but a maintainer should confirm that the model isn't tempted to "consume" a `[x]` checkbox-shaped status line and mark it complete in a follow-up turn.
4. `Todo.Info` type pulled from `./todo` at `:24` — confirm it's the persisted shape, not the wire shape (status enum vs string). `cleanAnchorLine(todo.status)` will silently render whatever the `.toString()` is.

No blockers — anchor design is sound, idempotency is guarded, test coverage matches the three failure modes named in the PR body.
