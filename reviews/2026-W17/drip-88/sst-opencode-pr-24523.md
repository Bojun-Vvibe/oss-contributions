---
pr: 24523
repo: sst/opencode
sha: 1c450b625cba98a083fd8516812ecf0d6df6e42d
verdict: merge-after-nits
date: 2026-04-27
---

# sst/opencode #24523 — fix(session): compare message positions instead of IDs in SessionPrompt.run

- **Author**: alfredocristofano
- **Head SHA**: 1c450b625cba98a083fd8516812ecf0d6df6e42d
- **Size**: +19/-6 in `packages/opencode/src/session/prompt.ts`. Closes #23490.

## Scope

Replaces lexicographic `id` comparisons (`lastUser.id < lastAssistant.id`, `m.info.id <= lastFinished.id`) with array-index comparisons (`lastUserIndex < lastAssistantIndex`, `i > lastFinishedIndex`) in the prompt loop's "decide whether to exit" and "iterate user messages after the last finished assistant" branches. The motivating bug: custom (non-monotonic) message IDs broke the loop-exit because lexicographic ordering on arbitrary IDs has no relationship to transcript order. Surface area is exactly the `SessionPrompt.run` scan; behavior on default monotonic IDs is preserved.

## Specific findings

- `packages/opencode/src/session/prompt.ts:1322-1346` — the back-scan loop now tracks three parallel `(message, index)` pairs: `(lastUser, lastUserIndex)`, `(lastAssistant, lastAssistantIndex)`, `(lastFinished, lastFinishedIndex)`. Indices are seeded to `-1` and only set when the corresponding message is found. Correct shape — the seeded `-1` plus the existing `if (lastUser && lastFinished) break` guard prevents any "found assistant but never found user" branch from comparing against a stale `-1`.
- `packages/opencode/src/session/prompt.ts:1366` — the exit predicate becomes `lastUserIndex < lastAssistantIndex`. This is the right invariant: "the most recent user message appears earlier in the transcript than the most recent assistant message" → the model already responded → safe to exit. Equivalent semantics to the old `lastUser.id < lastAssistant.id` only when IDs are time-monotonic; the new form is correct unconditionally.
- `packages/opencode/src/session/prompt.ts:1471-1474` — the `for...of` that previously used `m.info.id <= lastFinished.id` as a skip-condition becomes `for (let i = lastFinishedIndex + 1; i < msgs.length; i++)`. Strictly better: O(n - lastFinishedIndex) instead of O(n) per iteration with a comparison, *and* correct under non-monotonic IDs.
- **Missing test.** The bug class is "custom message IDs break loop exit"; the obvious regression test is a fixture with at least one `user` then `assistant(finish)` then `user` then `assistant(finish)` where `assistant.id < user.id` lexicographically (e.g., user IDs prefixed `u_z…` and assistant IDs prefixed `a_a…`). With the old code, the loop would exit too early or skip user replies; with this PR it should not. No such fixture appears in the diff.
- Minor: `lastFinishedIndex = i` is set inside the `if (!lastFinished && msg.info.role === "assistant" && msg.info.finish)` branch but `lastFinished` itself is never reassigned afterward in the loop. Fine, but a one-line comment "indices are stable once set; we walk msgs from the end and stop at first match" would make the invariant obvious to the next reader.

## Risk

Low. Net behavior change is "ordering predicates that previously assumed monotonic IDs now use actual array positions", which is strictly more correct. The skip-loop rewrite at `:1471` is also a small perf win. No new failure modes added.

## Verdict

**merge-after-nits** — add (1) a regression test exercising non-monotonic message IDs against the loop-exit branch, and (2) a one-line comment on the new index variables. Both are small. Core diff is right.
