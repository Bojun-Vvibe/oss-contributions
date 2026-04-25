# continuedev/continue #12151 — fix(gui): ignore stale provider model fetch results after provider switch

- **PR:** https://github.com/continuedev/continue/pull/12151
- **Head SHA:** `9deaa89bc950e0faaaca18444bcea8601c99314e`
- **Files changed:** 2 — `gui/src/forms/AddModelForm.tsx` (+15/−8), `gui/src/forms/AddModelForm.dynamicFetchRace.test.tsx` (+207, new).

## Summary

Fixes #12150 — a "late fetch wins" race in the AddModelForm. The dynamic-model fetch flow added in #12046 closes over `selectedProvider` inside the `handleFetchModels` callback, so a fetch initiated for provider A could resolve after the user had switched to provider B and write A's models into B's `fetchedModelsList`. The fix uses two refs: `selectedProviderRef` to identify the currently-selected provider at resolution time (instead of the stale closure value), and `fetchGenerationRef` to guard `setIsFetchingModels(false)` so a late spinner-clear from generation N doesn't stop the spinner for the user-initiated fetch in generation N+1.

## Line-level call-outs

- `gui/src/forms/AddModelForm.tsx:52-53` — `const selectedProviderRef = useRef(selectedProvider.provider); const fetchGenerationRef = useRef(0);`. Two-ref pattern is the right shape for this race. The provider ref encodes "what is current?" and the generation ref encodes "is this resolution still authoritative?". Both are needed: provider-equality alone doesn't catch the "user clicks B → switches back to A → first A-fetch resolves and clobbers second A-fetch" sub-race; the generation ref does. Good.
- `:60-63` — the `setSelectedProvider` effect now also bumps `fetchGenerationRef.current += 1` and `setIsFetchingModels(false)`. Resetting the spinner here is right; otherwise the user sees a stale spinner from the previous provider's still-in-flight request. Subtle: this means a fast double-switch (A→B→A) bumps the generation twice, which is fine — every comparison at `:88` is `===` against `fetchGeneration` snapshotted at *call* time, so each in-flight fetch correctly knows "I'm not the current generation" once another switch has happened.
- `:69-70` — `const fetchGeneration = fetchGenerationRef.current + 1; fetchGenerationRef.current = fetchGeneration;`. This is `++ref` written explicitly. Marginally clearer to write `const fetchGeneration = ++fetchGenerationRef.current` but functionally identical. No issue.
- `:78-80` — `if (selectedProviderRef.current === providerAtFetchTime) { setFetchedModelsList(models); }`. Correct guard. Note the asymmetry vs. the deleted `setFetchedModelsList((prev) => sameProvider ? models : prev)` — the new form uses the ref *outside* the setState callback, which means it reads the latest ref value at *resolution time* rather than at *commit time*. That's actually subtly better: between resolution and commit, React can't batch in a provider switch (effects run after commit), so reading the ref at resolution time is the correct snapshot.
- `:84-86` — the `finally` block guards `setIsFetchingModels(false)` behind `fetchGenerationRef.current === fetchGeneration`. Pairs with the eager `setIsFetchingModels(false)` in the provider-switch effect at `:62`. Together they ensure: (a) the spinner clears immediately on switch, (b) a late finally from the previous generation can't re-clear it (no observable difference, but cleaner) and crucially can't *accidentally re-set it* if any future refactor flips the order of operations.
- `:138-140` — `formMethods.setValue("apiKey", "")` now runs unconditionally on provider switch instead of only when `!RequiresApiKey`. That's a semantic change worth calling out separately — it means switching providers always wipes the typed API key, even when the new provider also requires a key. The third test (`releases the previous provider fetch lock…`, `:151-213`) implicitly covers the new behaviour (it types a fresh key after the switch). Reasonable change — clearing on switch is consistent with the "stale data" theme of the PR — but it's a separate concern from the fetch race and arguably belongs in its own PR with its own discussion.
- `:212` — `selectedProviderRef.current = match.provider` is set *inside* the `setSelectedProvider` callback before React commits. That ordering is intentional: the next call to `handleFetchModels` (synchronous follow-up) will see the new provider value. Good. Worth a one-line comment naming why the ref must lead the state.
- `gui/src/forms/AddModelForm.dynamicFetchRace.test.tsx:1-213` — three tests:
  1. Stale-fetch leak (`:83-123`) — initiates an OpenAI fetch, switches to Anthropic, resolves the OpenAI fetch, asserts no OpenAI models leak into Anthropic's listbox. Direct reproducer of #12150.
  2. API-key clearing on switch (`:125-149`) — covers the unrelated semantic change at `:138-140`.
  3. Lock release (`:151-213`) — hardest one, asserts that after switching providers a new fetch can fire immediately and that the old one's late resolve doesn't clobber the new one. Catches the regression where `isFetchingModels` would otherwise stay `true` and disable the fetch button.
  All three use a `deferred()` helper to control resolution order — exactly the right tool for race tests.

## Verdict

**merge-as-is**

## Rationale

Tight diagnosis, well-scoped two-ref fix that handles all three sub-races (stale write, double-switch, late finally), and a test file that exercises each one with explicit promise control rather than `setTimeout` flake. The unrelated unconditional API-key clear at `:138-140` is the only thing I'd ask to be split out, and even that is defensible under the same theme. The two-line "code comment" nits I called out are optional polish; nothing here is blocking. Land it.
