# block/goose#9048 — fix(goose2): keep model picker selections in sync

- URL: https://github.com/block/goose/pull/9048
- PR: #9048
- Author: matt2e (Matt Toohey)
- Head SHA: `83ff4c2f53893adbb66bbf4722e3837c250d9143`
- State: OPEN  | +1522 / -329

## Summary

Closes the "picker briefly shows Goose then snaps to user's saved choice on relaunch" jitter via four compounding fixes: (1) `resolveSelectedAgentId` no longer falls back to the `"goose"` literal during catalog-not-loaded windows — it preserves the persisted provider until validation completes; (2) `agentStore.setProviders` only persists a fallback choice when the incoming list is `validated`; (3) async `prepareSession`/`setModel` paths get monotonic version refs so a stale `await` that finishes after a newer user click is dropped; (4) `AppShell.ensureHomeSession` re-reads `useAgentStore.getState().selectedProvider` after every async hop instead of capturing it at callback creation.

## Specific references

- `app/AppShell.tsx:218-220`: new `currentProvider = () => useAgentStore.getState().selectedProvider ?? "goose"` closure invoked at every async boundary in the function (replaces the captured `agentStore.selectedProvider` from the outer scope).
- `AppShell.tsx:238-243`: post-`acpPrepareSession` re-resolution — if `liveProvider` differs from what was captured before the await, `resolvedProviderId = liveProvider`, otherwise the preference's `providerId` wins.
- `AppShell.tsx:248-254`: same re-resolution gate before `acpSetModel`, plus the conditional `sessionModelPreference.modelId && resolvedProviderId === sessionModelPreference.providerId` guard so a model-id from the *old* provider doesn't get applied to the *new* one.
- `app/hooks/useAppStartup.ts:62,76,116,140`: new `validated` boolean propagated through `applyProvidersFromInventory` → `setProviders(..., validated)`. Set `true` only after `loadProviderCatalog` / `loadProvidersAndInventory` complete with non-empty entries.
- `features/agents/stores/agentStore.ts:6` (referenced by PR — full diff not visible): `setProviders` now respects the new `validated` arg.
- `features/agents/stores/__tests__/agentStore.test.ts:206-250`: new describe block — `localStorage`-backed test asserting `setProviders([], false)` (unvalidated) does *not* overwrite the stored `claude-acp` choice; correlates with the `selectedProvider: "claude-acp"` setup at `:208`.
- `features/chat/lib/__tests__/agentProviderResolution.test.ts`: 148 new lines covering the resolver. (Full content not in window.)
- `features/chat/hooks/useResolvedAgentModelPicker.test.ts`: 117 new lines covering the picker hook's resolution path.

## Concerns / nits

- **Re-read pattern is correct but verbose**: 4 sites in `AppShell.tsx` now do the `liveProvider !== captured ? liveProvider : preference.providerId` triage. Worth extracting `resolveProviderForSession(captured, liveProvider, preference)` so the contract is named once and the assertion `resolvedProviderId === sessionModelPreference.providerId` (used twice as a gate) is centralized.
- The PR title says "model picker" but the load-bearing fix is `resolveSelectedAgentId` + `setProviders` validation gating in the *agent store* — the picker is downstream of both. Worth retitling for git-log searchability (e.g. "fix(goose2): preserve persisted provider during catalog hydration").
- The `validated = false` default at `useAppStartup.ts:62` is implicit and easy to forget at future call sites. Make it required (no default) so future contributors are forced to think about it.
- No test exercising the *monotonic version ref* contract — the PR body mentions it as fix #3 but the visible diff doesn't show the version-ref code. Confirm there's a "stale async finishes after newer click → discarded" test in the new 117-line `useResolvedAgentModelPicker.test.ts`.
- Co-Authored-By trailer mentions an LLM; that's a project convention question, not a blocker.

## Verdict

**merge-after-nits (man)** — correctly diagnoses the multi-source-of-truth problem and the four-pronged fix is sound; needs minor extract-helper polish and a confirmation that the version-ref contract is tested.
