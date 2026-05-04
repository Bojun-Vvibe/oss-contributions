# google-gemini/gemini-cli#26452 — fix(core): Fix hysteresis in async context management pipelines

- **Head SHA**: `2466d4b46ed640a2684b0fe36f6296607d2df91f`
- **Verdict**: `merge-after-nits`

## Summary

Adds hysteresis to the async context-management pipeline so background consolidation (snapshot) runs don't fire turn-by-turn. +314/-87 across context config (`profiles.ts`, `schema.ts`, `types.ts`), `contextManager.ts`, `pipeline/orchestrator.ts`, and a new `system-tests/hysteresis.test.ts`. Fixes #26451.

## Findings

- `packages/core/src/context/config/types.ts:32-36`: introduces optional `coalescingThresholdTokens?: number` on `ContextBudget`. Optionality is correct — older profiles continue to work. Good JSDoc.
- `packages/core/src/context/config/profiles.ts:81`: `generalistProfile` sets `coalescingThresholdTokens: 5000`. Reasonable default given the bumps below.
- `:121,128`: `nodeThresholdTokens` 1000→3000 and `maxTokensPerNode` 1200→4000. These are *significant* tuning changes bundled with the hysteresis fix. Consider splitting; if a future bisect blames this PR it'll be ambiguous whether the regression is from hysteresis logic or the new thresholds. Not a blocker but please call this out in the changelog.
- `packages/core/src/context/contextManager.ts:33`: `private lastTriggeredDeficit = 0;` — the entire hysteresis state. Single number is fine, but it's not reset anywhere on profile reload or session boundary. If a user switches profiles mid-session (rare but supported) the threshold will be evaluated against a stale deficit. Worth at least a TODO.
- `:82`: `this.evaluateTriggers(new Set())` added inside the post-render hook. Confirm this isn't going to re-enter `evaluateTriggers` recursively if a trigger itself emits a render event — the empty `Set()` argument suggests "no targets," which I read as a noop-by-design, but the name is misleading.
- `system-tests/hysteresis.test.ts` and `simulationHarness.ts`: nice — sim-based test for a state-machine bug is the right move. Spot-check that the harness asserts trigger *count* over N turns, not just a final-state property.

## Recommendation

Land after: (1) splitting or at least flagging the threshold bumps in release notes, (2) a TODO/reset for `lastTriggeredDeficit` on profile reload. Architecture is sound.
