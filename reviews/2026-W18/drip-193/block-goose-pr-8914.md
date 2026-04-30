# block/goose #8914 — feat: update provider row after saving credentials

- **URL:** https://github.com/block/goose/pull/8914
- **Head SHA:** `9ff2e05df8ac3f71853705d8ea2277cd572ad8dd`
- **Merge SHA:** `59c5693f7a1ad3f584241aa95d7938c57037c778`
- **Files:** `ui/goose2/src/features/settings/ui/ModelProviderRow.tsx` (5 small edits removing the `preserveSetupLayout` state machine: drops state declaration at `:81`, drops setter calls at `:172, 196, 311`, swaps `setShowSavedState(true) + setPreserveSetupLayout(true)` for `setShowSavedState(false)` at `:295-296`, and simplifies the render guard at `:401` from `hasFields && isConnected && !preserveSetupLayout` to `hasFields && isConnected`), `ui/goose2/src/features/settings/ui/__tests__/ModelProviderRow.test.tsx` (+55-line regression `"switches from setup save to connected controls after first configuration"` at `:113-167`)
- **Verdict:** `merge-as-is`

## What changed

Removes the `preserveSetupLayout` local-state machine that was holding the "setup panel" open after a successful first save. The previous shape kept `preserveSetupLayout=true` after save so the user saw a "Saved" confirmation in the same panel, but that conflicted with the natural state transition from `not_configured → connected`: once the row reports `isConnected`, the user expects to see the connected-row controls (Disconnect, Edit), not the setup form they just submitted.

The fix:

1. **Delete the `preserveSetupLayout` state at `:81`** — one fewer state slot to reason about.
2. **At the save success site `:295-296`** — flip from `setShowSavedState(true); setPreserveSetupLayout(true)` to `setShowSavedState(false)`. The transient "Saved" UI is dropped; the row immediately reflects the new `connected` status from the upstream `loadConfig()` call at `:294`.
3. **At the render site `:401`** — drop the `&& !preserveSetupLayout` guard so the connected-fields panel renders as soon as `hasFields && isConnected` is true.
4. **At the three other state-reset sites (`:172, 196, 311`)** — drop the now-unused setter call.

The 55-line regression test at `:113-167` is the proof: it renders a `SetupSaveRow` wrapper that flips the provider's `status` from `"not_configured"` to `"connected"` inside the test's `onSaveFields` callback, types an API key, clicks Save, then asserts via `waitFor`:

- `Disconnect` button is present (proves the connected-row layout rendered)
- `Edit` button is present (proves the editable controls are mounted)
- `Saved` button is **not** present (the negative assertion that locks out the regression — the previous behavior would have shown a "Saved" confirmation that this PR removes)

## Why it's right

- **Removes a state slot that was holding behavior-against-status-truth.** The provider's `status` field is the canonical truth for "is this provider configured." Holding `preserveSetupLayout=true` after a successful save meant the UI was rendering the setup form even though `isConnected` was true — the local state was overriding the source of truth. Deleting the override means the UI reflects the canonical status, which is the right shape for any settings panel.
- **The render guard simplification at `:401` is unambiguously cleaner.** `hasFields && isConnected` is a two-condition predicate that maps directly to the user's mental model ("this provider has fields, and it's connected, so show the connected panel"). The previous three-condition `hasFields && isConnected && !preserveSetupLayout` carried an implementation detail (the local state machine) into the render predicate.
- **Negative assertion in the test (`expect(screen.queryByRole("button", { name: /saved/i })).not.toBeInTheDocument()` at `:163-165`) locks out the previous behavior.** This is the right shape — without the negative, a future "let's bring back the Saved confirmation" change would silently pass the test while regressing the UX this PR is fixing.
- **The test wraps `ModelProviderRow` in a `SetupSaveRow` stateful wrapper at `:117-149`** so the `status` flip from `"not_configured"` to `"connected"` happens inside the same render tree the user sees — that's a higher-fidelity test than mocking the status update, because it exercises the actual prop-change → re-render → guard-re-evaluate path that the production code goes through after `loadConfig()`.
- **Five small surgical edits, no drive-by changes.** Every removal is the same identifier (`preserveSetupLayout`) at known sites — no refactor sprawl, no opportunistic cleanup, nothing for a reviewer to second-guess.

## No nits

The fix is minimal, the test locks the new behavior with both positive (Disconnect/Edit present) and negative (Saved absent) assertions, the render guard simplifies in the right direction, and the change preserves the established `setShowSavedState` setter for any future "show transient confirmation" work without keeping the state machine alive in the meantime. Ship it.
