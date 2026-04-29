# QwenLM/qwen-code #3735 — fix(core): auto-compact subagent context to prevent overflow

- **PR:** https://github.com/QwenLM/qwen-code/pull/3735
- **Head SHA:** 07480e3db2524b24070d4f35b8901e9c2740f0c9
- **Files changed:** 8 files, +1283 / −1020
- **Verdict:** `merge-after-nits`

## What it does

Subagent chats now share the main agent's compaction trigger: when history grows past
the configured threshold, `GeminiChat.tryCompress` runs in the chat layer for both
main and sub agents. The eager pre-compaction call in the main session loop is
removed (replaced by the chat-layer trigger), and a new `compressed` stream event is
surfaced to subagent runtime so subagent compactions appear in the same debug log
stream as main-session compactions.

Key sites:
- `agents/runtime/agent-core.ts:543-552` — adds the `streamEvent.type === 'compressed'`
  branch with a `[AGENT-COMPACT] subagent=... round=... tokens X -> Y` debug line.
- `core/client.test.ts:407-415` — every test now defaults `tryCompressChat` to a NOOP
  spy so existing hand-rolled chat mocks (which don't implement `tryCompress`) don't
  blow up; tests that exercise compression directly override the spy.
- `core/client.test.ts:240-244` — the `findCompressSplitPoint` "trailing in-flight
  functionCall" test changes its expected split point from `2` to `3` with an updated
  comment ("the in-flight fallback compresses everything except the trailing fc").
  This is a real semantic change to the split heuristic that the PR body doesn't call
  out clearly enough.

## What's good

- The PR reports a concrete before/after number ("Subagent rounds before 400: 5 →
  ran until session ended naturally"); that's a substantive validation, not vibes.
- Acknowledges the `sessionTokenLimit` ordering trade-off explicitly (gate now reads
  the previous turn's prompt count, so a session sitting just over the limit may trip
  one turn earlier). Calling that out as intentional is the correct disclosure shape.
- The debug log line `[AGENT-COMPACT] subagent=${this.subagentId} round=${turnCounter}`
  gives operators a single grep target across both surfaces.

## Nits / risks

1. **The `findCompressSplitPoint` heuristic change is a behavior shift.** Returning
   `3` instead of `2` when the trailing turn is `model+functionCall` means more
   history gets compressed away on every interrupted-tool-call retry. Worth pinning
   with a separate test that exercises the *new* "in-flight fallback" path with a
   second history shape (e.g. trailing user message after a model+fc) to prove the
   fallback only fires on the in-flight case.
2. The default-NOOP `tryCompressChat` spy in `client.test.ts:411-415` silences the
   `compress was called` assertion path in every existing test that didn't opt in.
   That's pragmatic but it means a future regression where compression *should* fire
   and silently doesn't will pass the suite. Consider asserting `mockTryCompress.
   mock.calls.length === 0` in any test claiming "no compaction".
3. The PR removes "the eager pre-call in the main session loop" — confirm in the PR
   body which file/line that was, so reviewers can verify it's actually gone and not
   just guarded behind a different condition.
4. The `[AGENT-COMPACT] subagent=...` log uses `this.subagentId` — confirm this
   value is non-empty for the main agent run too (or use a sentinel like `"main"`),
   otherwise grepping the log produces an empty `subagent=` field for half the rows.

## What I learned

Moving compaction from the session-loop pre-call into the chat layer is the right
unification (both surfaces now share one trigger), but the hidden cost is that every
existing chat-mock test now has to opt in to a `tryCompress` shape. The PR's
default-NOOP spy is a reasonable migration step; the long-term answer is a small
`MockChat` test helper.
