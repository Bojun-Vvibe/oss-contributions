# cline/cline PR #10319 — fix: Model metadata fails to update when change model

- **PR:** https://github.com/cline/cline/pull/10319
- **Author:** water-in-stone
- **Head SHA:** `77f465c67b12bd61c5e6e2de106fbecee53b18ff`
- **Files:** 2 (+53 / -12)
- **Verdict:** `merge-after-nits`

## What it does

Fixes #10248. In Settings → ClineModelPicker, clicking a model card from
the "Recommended" or "NEW" chips updated the displayed model name
immediately but the price metadata (input/output/cache prices) lagged
behind, often showing "Free" for paid models because the previous selection's
metadata stayed pinned in the React state. Users were making cost decisions
on stale data.

The fix has two parts:

1. **Don't write `undefined` over a valid persisted `clineModelInfo`** in
   `handleModelChange` when the freshly-clicked model isn't in the
   `clineModels` map yet (newly launched models served from a separate
   endpoint).
2. **Use `searchTerm` as the effective model id while waiting for the
   gRPC round-trip** in the `selectedModelInfo` useMemo, so the price view
   updates synchronously with the click instead of after the
   `apiConfiguration` arrives back.
3. **Surface a loading state** (`isModelInfoLoading`) when the model
   isn't in `clineModels` and gRPC hasn't confirmed the selection — so
   `ModelInfoView` can show "Loading…" instead of stale prices.

## Specific reads

- `webview-ui/src/components/settings/ClineModelPicker.tsx:198-218` —
  `handleModelChange` extracted `modelInfo = clineModels?.[newModelId]`
  before the call to `handleModeFieldsChange`, then conditionally
  spreads `clineModelInfo` only when defined. Spread-when-defined is
  the right shape — `{ ...(modelInfo != null ? { clineModelInfo:
  modelInfo } : {}) }`. The detailed comment explains why writing
  `undefined` was the bug. **Good defensive comment** that ties the
  code to the real-world failure mode.
- `webview-ui/src/components/settings/ClineModelPicker.tsx:222-260` —
  the rewritten useMemo. New variables `configModelId`,
  `isPendingGrpcUpdate`, `effectiveModelId`, `modelInfoFromClineModels`,
  `effectiveModelInfo`, `isModelInfoLoading`. Clean, every variable
  pulls weight. The decision matrix is explicit:

  ```
  isPendingGrpcUpdate = !!searchTerm && searchTerm !== configModelId
  effectiveModelId = isPendingGrpcUpdate ? searchTerm : configModelId
  effectiveModelInfo = clineModels?.[effectiveModelId] ?? selected.selectedModelInfo
  isModelInfoLoading = isPendingGrpcUpdate && !modelInfoFromClineModels
  ```

  This handles all four combinations of (in-cache, not-in-cache) ×
  (gRPC-confirmed, gRPC-pending) correctly. The free-model case
  (`freeClineModelIdSet.has(...)`) hard-zeros the prices, so even in
  the "in-cache, just-clicked" state a free model never momentarily
  displays a non-zero price.
- `useMemo` deps now include `clineModels` and `searchTerm` (was
  `[apiConfiguration, currentMode, freeClineModelIdSet]`). Correct —
  any change to either should re-derive. Watch out for stable
  reference assumption on `clineModels`: if it's a new object on
  every parent render, this useMemo provides no caching benefit. Worth
  a quick check upstream.
- The `isModelInfoLoading` field is added to the destructured return
  but I can't see in the visible diff how `ModelInfoView` consumes it
  — the PR claim is "displays Loading… instead of stale prices" but
  the consumer-side change isn't in the first 80 lines. Need to verify
  `ModelInfoView` actually checks the new prop.

## Risk surface

**Low-medium.** Pure UI-state logic, no protocol/persistence change.
Three things to watch:

1. **The `searchTerm !== configModelId` heuristic** assumes
   `searchTerm` is *only* set as a side-effect of `handleModelChange`.
   If `searchTerm` can be set via a free-text search input (the name
   suggests it can), then typing in the search box would trigger
   `isPendingGrpcUpdate = true` and `effectiveModelId` = whatever the
   user typed, even if they haven't clicked anything. That's
   incorrect — partial typed text shouldn't be treated as a model
   selection. Need to confirm `searchTerm` is only ever set to a
   complete, valid model id.
2. **`modelInfoFromClineModels ?? selected.selectedModelInfo`** — when
   the effective id isn't in `clineModels`, fall back to the
   config-stored info. But the config-stored info might still be
   stale (it's the *previous* model's info if the gRPC round-trip
   hasn't completed yet). The PR's `isModelInfoLoading` flag
   compensates *visually* by showing "Loading…", but if any other
   component reads `selectedModelInfo` directly without checking
   `isModelInfoLoading`, it'll still see stale data. Worth a grep.
3. **`useMount(() => …)`** at the bottom — the diff cuts off there.
   Need to see if it's affected.

## Suggestions before merge

- Confirm `searchTerm` is only set on full model selection, not on
  free-text search input. If it can be set on partial input, gate the
  `isPendingGrpcUpdate` check on `clineModels?.[searchTerm] != null`
  to avoid treating "gpt-" as a pending selection.
- Verify `ModelInfoView` consumes `isModelInfoLoading` correctly —
  the PR description claims this but the consumer-side diff isn't
  visible.
- Grep for other readers of `selectedModelInfo` in the picker tree
  to confirm none read it without the loading flag.
- Add a webview integration test: click a paid model from the
  Recommended list, assert that the price area shows "Loading…"
  briefly then the correct paid prices, never "Free."

Verdict: merge-after-nits — the underlying logic is correct and the
inline comments are excellent (rare for a UI fix to clearly explain
the gRPC-lag race condition). The two unverified consumer-side
assumptions are not blockers but should be confirmed before merge.

## What I learned

The "synchronous UI state vs async server state" mismatch is the
classic React/RPC bug. Two patterns help:

1. **Optimistic local state** (what this fix does) — derive a
   transient `effectiveX` that prefers the just-clicked value over
   the still-fetching server value, plus a `isLoading` flag for
   anything that can't be derived locally.
2. **Avoid writing `undefined` over valid persisted state** — if
   you don't have fresh data, *don't write at all*, rather than
   writing a sentinel that destroys the previous good value.

The bug-symptom pattern — "showing 'Free' for a paid model" — is
particularly nasty because there's no error, no spinner, no
exception. It's quietly *wrong*, in a direction that costs money.
That's the highest-stakes class of UI bug, and a good reminder that
"showing the previous value while loading the next" is a default
that's not always safe.
