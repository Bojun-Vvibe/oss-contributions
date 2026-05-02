# google-gemini/gemini-cli #26357 — fix(core): Minor fixes for generalist profile

- URL: https://github.com/google-gemini/gemini-cli/pull/26357
- Head SHA: `10559112753985647f810ea039b1931453cfe28b`
- Author: @joshualitt
- Stats: +37 / -14 across 7 files

## Summary

Three loosely-related tweaks to the context-management "generalist"
profile: (1) introduces a new `coalescingThresholdTokens` budget knob
to suppress turn-by-turn snapshot churn, (2) raises per-node distillation
/ truncation thresholds from 1000/1200 → 3000/4000 tokens, and (3)
gates a new `scrubHistory` pass on `isContextManagementEnabled()` in
`geminiChat.getHistory`. Also a minor snapshot-prompt reword.

## Specific feedback

- `packages/core/src/context/config/profiles.ts:81` — `coalescingThresholdTokens:
  5000` lands as the default for the generalist profile. Combined with
  the consumer at `contextManager.ts:144-160`, this means consolidation
  is **suppressed** unless the over-budget deficit is ≥ 5k tokens. That
  is the intent, but it materially changes the eviction cadence — the
  PR description should call out the cost/quality tradeoff (fewer
  utility-model calls, but stale context lingers ~5k tokens longer).
- `packages/core/src/context/config/profiles.ts:121-128` —
  `nodeThresholdTokens: 1000 → 3000` and `maxTokensPerNode: 1200 →
  4000` are both >2× their previous values. These are profile-level
  defaults so any existing user override in their own
  `gemini.config.json` is preserved — but anyone relying on the
  *default* will see noticeably different summarization behaviour
  starting at this commit. Worth a CHANGELOG entry.
- `packages/core/src/context/contextManager.ts:144-160` — the
  `if (targetDeficit >= threshold)` guard correctly defaults to `0`
  when `coalescingThresholdTokens` is unset (`|| 0`), so older configs
  / non-generalist profiles see no behavioural change. Good.
- `packages/core/src/context/contextManager.ts:144-150` — the
  `garbageCollectCache` call is now **inside** the `if (targetDeficit
  >= threshold)` block. Previously the cache was GCd unconditionally
  whenever any nodes aged out. That means under the new threshold, the
  token-calculator cache will accumulate references to evicted nodes
  for longer. Verify this isn't a memory-growth regression in
  long-running sessions; consider GC-ing on a separate (lower)
  cadence than the consolidation event.
- `packages/core/src/core/geminiChat.ts:776-784` — `scrubHistory(
  [...history])` is called only when `isContextManagementEnabled()`.
  The diff doesn't show `scrubHistory`'s implementation; reviewer
  should verify it doesn't drop tool-call IDs or thought-parts that
  downstream consumers (resume, telemetry) depend on. The conditional
  gating is correct: pre-existing callers that haven't opted into
  context management will continue to see raw history.
- `packages/core/src/core/client.test.ts:+1` — only adds a
  `isContextManagementEnabled: vi.fn().mockReturnValue(false)` mock.
  Confirm the parallel test for the `true` branch exists elsewhere; if
  not, the new conditional code path is untested.
- `packages/core/src/context/utils/snapshotGenerator.ts:20-22` — minor
  prompt rewording, splitting the "discard filler" sentence onto its
  own line. Semantically identical, slightly clearer instructions.
  No risk.
- `packages/core/src/context/config/schema.ts:+5` and `types.ts:+5`
  — schema and TS-type additions are mirror-image. No drift.

## Verdict

`merge-after-nits` — three small fixes bundled together; each
defensible. The user-facing impact (snapshot cadence + per-node token
ceilings) deserves a CHANGELOG line. Verify (a) no memory-growth
regression from delayed `garbageCollectCache`, and (b) the `true`
branch of the new `isContextManagementEnabled` gate has test coverage.
