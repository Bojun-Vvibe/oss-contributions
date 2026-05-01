# PR #26332 — fix(acp): resolve agent mode disconnect and improve mode awareness

- Repo: google-gemini/gemini-cli
- Head: `ae1549df7c3ebc67b091c8f3eaaf93d25aa26ae4`
- URL: https://github.com/google-gemini/gemini-cli/pull/26332
- Verdict: **merge-after-nits**

## What lands

Fixes a long-standing ACP bug: the agent's `ApprovalMode` (Plan / Default /
YOLO / Auto-Edit) was set on `Config` but never propagated to (a) the system
prompt the LLM sees, or (b) the ACP client UI. Result: an external ACP host
(JetBrains, Zed) could flip the mode and the agent would silently keep
operating under the old mode, and any internal mode change wouldn't update
the host. The fix wires a new `CoreEvent.ApprovalModeChanged` event through
both surfaces.

## Specific findings

- `packages/core/src/config/config.ts:2690` adds the emit at the only call
  site that mutates approval mode (`Config.setApprovalMode`):
  ```
  this.policyEngine.setApprovalMode(mode);
  this.refreshSandboxManager();
  coreEvents.emitApprovalModeChanged(this.getSessionId(), mode);
  ```
  This is the right chokepoint — the policy-engine and sandbox refresh both
  happen synchronously before the emit, so listeners observing the event
  see consistent state.
- `packages/cli/src/acp/acpSession.ts:75-89` listens in the per-session
  constructor and filters by `payload.sessionId === this.id`. Correct
  scoping — multiple ACP sessions in the same process won't cross-emit.
- `packages/core/src/core/client.ts` (visible in diff snippet) listens in
  `GeminiClient` to refresh the system prompt; the snapshot diffs in
  `packages/core/src/core/__snapshots__/prompts.test.ts.snap` confirm the
  preamble now contains `You are currently operating in **Plan** mode.` /
  `**Default** mode.` etc. across all 19 prompt variants.

## Issues to address before merge

- **Listener leak risk.** `acpSession.ts:75-79` registers
  `coreEvents.on(CoreEvent.ApprovalModeChanged, this.handleApprovalModeChanged)`
  in the constructor but I don't see a corresponding `coreEvents.off(...)`
  in any session-disposal path. Long-running processes that create and
  destroy sessions (e.g. JetBrains plugin lifecycle) will accumulate dead
  listeners and the event fan-out grows unboundedly. Either add
  `coreEvents.off()` in a `dispose()` / `cancel()` path or use a `once`
  pattern keyed on session lifetime.
- **`agent_message_chunk` carries `[MODE_UPDATE] ${mode}` as a string
  payload.** This works but it's ad-hoc — ACP clients have to string-match
  on the prefix to distinguish a mode update from real model output. ACP
  has `sessionUpdate.mode_changed` style events for exactly this. Worth
  asking upstream whether a typed update is preferable; the string
  approach will create migration pain later.
- **Snapshot churn is enormous (~19 snapshot diffs).** All correct, but
  the next time someone touches the system prompt preamble they'll have to
  re-snapshot 19 places. A separate `getModePreamble(mode)` helper with
  its own focused unit test would isolate the change.

## Verdict rationale: merge-after-nits

The wiring is correct end-to-end (config → event → both client and ACP
session) and the test additions at `acpSession.test.ts:567-587` and
`client.test.ts:2062-2082` directly assert the new behavior. The listener
leak is the only thing that could turn into a real bug — it's worth one
follow-up commit. The string-payload concern is a design conversation, not
a blocker.
