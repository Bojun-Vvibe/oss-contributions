# sst/opencode#24544 — fix(session): compare message positions instead of IDs in SessionPrompt.run

- PR: https://github.com/sst/opencode/pull/24544
- Head SHA: `1c450b62`
- Diff: +19 / -6 in `packages/opencode/src/session/prompt.ts`

## What it does
Closes #23490. Replaces lexicographic message-ID comparisons inside `SessionPrompt.run`'s loop-exit and pending-input-replay logic with array-index comparisons against the in-memory `msgs[]` snapshot. The bug: opencode's loop used `lastUser.id < lastAssistant.id` (`prompt.ts:1354`) and `m.info.id <= lastFinished.id` (`:1459`) to decide "has the user spoken since the assistant last finished?" — but if any message ID is custom-assigned (anything not strictly monotonic ASCII-sortable, e.g. UUIDs, externally-supplied IDs from the SDK, or any non-ULID-shaped string), lexicographic ordering does not reflect chronological transcript order. The wrong branch fires and the loop either exits when it should continue or replays already-processed user messages.

## Strengths
- Correct root-cause framing: message IDs are an *identifier* contract, not an *ordering* contract. The transcript order is `msgs[]` index, which the scan loop already walks (`for (let i = msgs.length - 1; i >= 0; i--)`). The fix piggybacks on that walk to capture `lastUserIndex`, `lastAssistantIndex`, `lastFinishedIndex` (`prompt.ts:1325-1342`) at zero extra cost.
- The replay loop replacement at `:1471-1473` is a strict improvement: instead of iterating *all* `msgs` and skipping with `m.info.id <= lastFinished.id`, it now starts at `lastFinishedIndex + 1`. Same result, O(N - finished) instead of O(N), and no comparison footgun.
- Initialization of all three indices to `-1` makes the comparison `lastUserIndex < lastAssistantIndex` correctly fall through when `lastUser` is missing (the boolean precondition `lastUser` already guards), and the `>= 0` semantics are obvious to a reader.

## Concerns
- **No test added.** The PR description says "the logic correctly handles custom message IDs by relying on array positions" but there's no regression test that constructs a message list with non-monotonic IDs (e.g. `["msg_zzz", "msg_aaa", "msg_mmm"]`) and asserts the loop-exit branch behaves correctly. Without it, a future refactor that re-introduces ID comparison silently regresses #23490. The existing `test/session/` suite has plenty of fixtures that could be extended — a single test with three messages (user with `id: "z"`, assistant-finished with `id: "a"`, user with `id: "m"`) would lock this down.
- The for-loop conversion at `:1471-1473` from `for...of` to indexed `for` loses the `for...of` ergonomics for a 3-line body. Functionally identical, but the diff is a touch noisier than necessary; an alternative would have been `msgs.slice(lastFinishedIndex + 1).filter(m => m.info.role === "user")...`. Minor stylistic call — current form is fine.
- The relationship between the three `*Index` variables and the existing three `last*` references is now duplicated state. They will always agree (set in the same `if` block), but a future contributor might update one without the other. A single helper like `findLastByPredicate(msgs, pred): {value, index}` would collapse this. Not blocking.
- Comment-worthy: the bug shape (custom message IDs breaking ordering assumptions) is subtle enough that a `// IDs are not ordered; use index for transcript order` line above the `lastUserIndex < lastAssistantIndex` comparison would save a reviewer in 2026-Q3 from re-deriving the lesson.

## Verdict
**merge-after-nits** — correct fix, correct framing, but the missing regression test for #23490 means the next refactor could silently re-introduce the bug. Add a 15-line test with non-monotonic IDs and merge.

## What I learned
"IDs sort meaningfully" is one of those invariants people assume holds because it usually does (auto-increment, ULIDs, snowflake IDs all sort chronologically by construction). The moment any external system can mint IDs into your transcript, the assumption breaks. The defense is: never reach for ID comparison when array index is right there.
