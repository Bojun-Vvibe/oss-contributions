# Review: QwenLM/qwen-code #3826

- **Title:** fix(cli): refine slash-like prompt routing
- **Head SHA:** `d3fe338f57369ddf7a7353da74eccdb19ad3d16d`
- **Size:** +569 / -65
- **Files touched (sample):**
  - `packages/cli/src/ui/AppContainer.tsx`
  - `packages/cli/src/ui/hooks/slashCommandProcessor.test.ts`
  - `packages/cli/src/ui/hooks/slashCommandProcessor.ts`
  - `packages/cli/src/ui/hooks/useGeminiStream.test.tsx`
  - `packages/cli/src/ui/hooks/useGeminiStream.ts`
  - `packages/cli/src/ui/hooks/useHistoryManager.ts`
  - `packages/cli/src/ui/hooks/useMessageQueue.test.ts`
  - `packages/cli/src/ui/hooks/useMessageQueue.ts`

## Critique

Reworks how slash-prefixed input is classified. Adds a `sentToModel` flag to user history items, exposes a new `SlashCommandProcessorActions` type, and changes the message-queue draining policy so that "plain prompts" are batched and slash commands pop one-at-a-time, preserving typed order.

Specific lines:

- `AppContainer.tsx:765` — passes `historyManager.updateItem` as a new (positional) arg to the slash-processor hook. Positional growth on a hook that already takes ~15 args is becoming unwieldy; consider migrating to an options object in a follow-up. Also, mock setup in `slashCommandProcessor.test.ts:188-192` had to add `undefined, mockUpdateItem` to keep tests compiling — confirms positional churn.
- `AppContainer.tsx:1144-1149` — user history item now constructed with `sentToModel: true` and the `satisfies HistoryItemWithoutId` assertion. Good — using `satisfies` instead of an `as` cast preserves inference. Verify `HistoryItemWithoutId` actually has `sentToModel` typed as required (otherwise the satisfies check is vacuous).
- `AppContainer.tsx:2245-2249` — comment changed from "Two-phase: batch plain prompts as one turn, else pop next slash command." to a longer explanation. Behavior matches: `plainPrompts.length > 0 ? plainPrompts.join('\n\n') : popNextSegment()`. The "join with \n\n" semantics mean two queued plain prompts become a single turn — confirm this is desired UX vs. distinct turns. There's a regression risk if a user typed two prompts expecting two responses.
- `slashCommandProcessor.test.ts:115-141` — extracted `createMockActions` helper, which is good test hygiene. The new mocks include `openArenaDialog`, `openManageModelsDialog`, `handleResume`, `openDeleteDialog`, `openExtensionsManagerDialog`, `openMcpDialog`, `openHooksDialog`, `openRewindSelector` — all `vi.fn()`. Implies the production `SlashCommandProcessorActions` type grew significantly; downstream consumers of the hook will need to supply all of these or the type will fail.
- New test at `slashCommandProcessor.test.ts:292-294` — "should let slash-prefixed file paths fall through to the model" — exactly the right regression test for this fix. The cut-off at line 150 means I can't see the assertion; spot-check that it asserts the slash command processor returns `false` (not handled) for paths like `/tmp/foo.txt` or `/Users/x`.
- 569 lines for a "refine" is substantial. Worth requesting a clearer PR description that calls out exactly which input shapes changed routing.

## Verdict

`merge-after-nits`
