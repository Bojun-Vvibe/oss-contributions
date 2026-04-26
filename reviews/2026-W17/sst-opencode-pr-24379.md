---
pr: 24379
repo: sst/opencode
sha: e8131301887b04668d72dfe5178cdc247f71f7bd
verdict: merge-as-is
date: 2026-04-26
---

# sst/opencode #24379 — fix(session): use transcript position instead of lexical ID compare in prompt loop

- **Author**: tiffanychum
- **Head SHA**: e8131301887b04668d72dfe5178cdc247f71f7bd
- **Closes**: #23490
- **Size**: ~127 diff lines across `packages/opencode/src/session/prompt.ts` and `packages/opencode/test/session/prompt.test.ts`.

## Scope

`SessionPrompt.run` decided two things via lexical compare on opaque message IDs:
1. **Loop-exit at `finish="stop"`**: `lastUser.id < lastAssistant.id`
2. **Post-finish reminder injection**: `m.info.id <= lastFinished.id`

The HTTP API explicitly accepts caller-supplied `messageID` on prompt input. When a caller supplies an ID like `msg_zzzzzzzz...` that lex-sorts greater than `MessageID.ascending()` output, both comparisons reverse: the loop runs an extra iteration after `finish="stop"` and sends a transcript ending in an assistant turn, and Anthropic-via-OpenRouter rejects with "last message must be a user message". This PR replaces the ID compare with array-position compare during the same backward walk that already collects these messages.

## Specific findings

- `packages/opencode/src/session/prompt.ts:1324-1351` — three new index trackers (`lastUserIndex`, `lastAssistantIndex`, `lastFinishedIndex`) initialized to `-1` and populated alongside the existing first-match pattern in the reverse walk. **No new traversals**, just additional bookkeeping in the same loop. Good — minimal impact on hot path.
- `packages/opencode/src/session/prompt.ts:1369` — the loop-exit predicate changes from `lastUser.id < lastAssistant.id` to `lastUserIndex < lastAssistantIndex`. The PR body's "canonical transcript order produced by `MessageV2.filterCompactedEffect` → `page()` → `ORDER BY time_created, id`" is the right justification — array position *is* the canonical order, ID compare was always a happens-to-work proxy.
- `packages/opencode/src/session/prompt.ts:1471-1474` — the post-finish reminder loop changes from `for (const m of msgs) { if (m.info.id <= lastFinished.id) continue; ... }` to `for (let i = lastFinishedIndex + 1; i < msgs.length; i++) { const m = msgs[i]; ... }`. This is also a small perf win — skips the strictly-earlier portion of the array entirely instead of iterating-and-continue.
- `packages/opencode/test/session/prompt.test.ts:342-407` — `loop exits when caller-supplied user messageID lex-sorts after assistant id` is a clean regression test. It uses `MessageID.make("msg_zzzzzzzzzzzzzzzzzzzzzzzz")` for the user (intentionally lex-greater than any hex-prefix `MessageID.ascending()` output), `MessageID.ascending()` for the assistant, and `time: { created: userTime + 1 }` to make the *transcript ordering* unambiguous via `time_created`. Asserts `result.info.role === "assistant"`, `result.info.finish === "stop"`, and crucially `expect(yield* llm.calls).toBe(0)` — confirming the loop exits *immediately* on the existing assistant turn rather than calling the LLM again.
- The test's PR-body-style comment on `time: { created: userTime + 1 }` ("Strictly greater than userTime so transcript ordering by time_created is unambiguous regardless of ID lex order") is exactly the right level of explanation — it documents that the test is valid *because* of the ORDER BY contract, not by accident.

## Risk

Very low. The fix is mechanical and behavior-preserving for OpenCode-generated monotonic IDs (where lex order matches array order). The single new test pins the regression. Performance is neutral-to-better.

## Verdict

**merge-as-is** — textbook bugfix: minimal change, single regression test that pins exactly the broken case, performance-neutral, and the rationale ("ID compare was a proxy for array order, switch to array order directly") is unambiguous. The HTTP-API caller-supplied-ID surface is exactly the hostile-input class this should have been written for from the start.
