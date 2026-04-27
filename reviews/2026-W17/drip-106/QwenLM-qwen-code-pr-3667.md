# QwenLM/qwen-code PR #3667 — fix(cli): refresh static header on model switch

- **Link**: https://github.com/QwenLM/qwen-code/pull/3667
- **Head SHA**: `426fa23c9d5c6c836c274c1199293d4aebcb8fb2`
- **Author**: pomelo-nwu
- **Size**: +324 / -20 across 5 files
- **Verdict**: `merge-after-nits`

## Files changed
- `packages/core/src/config/config.ts` — adds public `Config.onModelChange(callback): () => void` subscription API. The returned function is the unsubscribe handle (idiomatic listener pattern).
- `packages/core/src/config/config.test.ts` — covers the new pub/sub: subscribe → setModel → assert called once with new model; unsubscribe → setModel → assert not called.
- `packages/cli/src/ui/AppContainer.tsx:301-303, 415-431, 476-491` — three coordinated changes: (a) drops the `getCurrentModel = useCallback(...)` polling helper; (b) deletes the `useEffect` polling block (`setInterval(checkModelChange, 1000)`); (c) adds an event-driven `useEffect(() => { const unsub = config.onModelChange((model) => { setCurrentModel(prev => { if (prev === model) return prev; refreshStatic(); return model; }); }); return unsub; }, [config, refreshStatic])`.
- `packages/cli/src/ui/components/MainContent.tsx` — `AppHeader` stays inside Ink `<Static>`, but `currentModel` is removed from the static remount key (see test file for the contract).
- `packages/cli/src/ui/components/MainContent.test.tsx` (new, 246 lines) — pin tests for `<Static>` props/items spy + `appHeaderSpy` to assert that header re-render is driven by the explicit `refreshStatic()` call, not by a remount-key change.

## Analysis

This is the right architectural fix for a UX bug. The pre-fix flow:

1. Header renders inside Ink's `<Static>` region (which is append-only — output stays at the top of the scrollback forever).
2. Some prior PR added `currentModel` to the static-region remount key, so changing the model would force-remount the static region — which works, but **also re-prints the entire banner and history** at the top of the terminal, producing visible duplicate output.
3. To detect model changes at all, `AppContainer` polled `config.getModel()` every 1000ms via `setInterval` — guaranteeing at least 1s lag for the header to refresh after `/model`, and a per-second wakeup that defeats Node's idle batching.

The fix replaces both:
- **Subscription instead of poll**: `Config.onModelChange(cb)` is a normal pub/sub. The `useEffect` registers, returns the unsubscribe; React tear-down semantics handle cleanup. No interval, no lag.
- **Explicit refresh instead of remount-key**: when the callback fires, the new code calls `refreshStatic()` (Ink's escape hatch for clearing+remounting the static region) explicitly, *only* when the model actually changed (`if (prev === model) return prev;` guards against spurious notifications). Removing `currentModel` from the static remount key means the rest of the time, model state changes don't trigger any static-region churn.

The `setCurrentModel` callback shape is a nice touch:
```js
setCurrentModel((prev) => {
  if (prev === model) {
    return prev;       // no state change → React skips re-render
  }
  refreshStatic();     // side-effect ONLY on real change
  return model;
});
```
Doing the `refreshStatic()` side-effect inside the state-updater function is unconventional (React docs warn against side-effects in updaters because of Strict Mode double-invocation), but here it's the right shape because: (a) the alternative — comparing in the `useEffect` body — race-conditions with concurrent model changes if the subscription fires twice fast, (b) `refreshStatic` is idempotent enough that a Strict-Mode double-invocation just results in two clear-and-remount cycles which is visually indistinguishable from one, (c) it keeps "decide whether to refresh" and "actually refresh" co-located.

The `MainContent.test.tsx` is solid — 246 lines is a lot for a test file but the mocking shape is justified: `vi.mock('ink', ...)` to replace `<Static>` with a spy-instrumented passthrough, separate spies for `staticPropsSpy`, `staticItemsSpy`, `appHeaderSpy`, plus `vi.mock('./AppHeader.js', ...)` to render the version as `APP_HEADER:${version}` so a single text grep confirms re-render. That test will catch any future PR that re-introduces `currentModel` into the static remount key.

## Nits to address before merge

1. **Strict Mode double-invocation concern is unaddressed in code or comments**. Calling `refreshStatic()` inside the state updater means under React Strict Mode (which double-invokes state updaters in dev) the Ink static region will be cleared+remounted twice per real model change. This is harmless visually but produces two ANSI-clear sequences in scrollback during dev sessions. Either move the `refreshStatic()` to the body of the callback (before `setCurrentModel`) and accept the race-window risk, or add a one-line comment naming the tradeoff. My preference: the body-of-callback version is cleaner.
2. **No test pins the unsubscribe behavior at the AppContainer level**. The `config.test.ts` tests cover the core's pub/sub contract, but the `AppContainer` integration could regress if a future PR drops the `return unsub;` from the `useEffect`. A 5-line test that mounts `AppContainer`, fires a model change, unmounts, fires another model change, and asserts no error/no second update would lock the cleanup contract.
3. **The `Config.onModelChange` API doesn't fire an initial value**. That's the right default (subscribers shouldn't get retroactive history), but a doc comment naming this would help: subscribers are responsible for reading `config.getModel()` at registration time if they want the current value. The `useState(() => config.getModel())` initialization at line 303 implicitly handles this for AppContainer, but other future subscribers won't have the same context.
4. **The polling interval was 1000ms**, which means model-change-detection latency drops from ≤1s to ~immediate. That's a measurable UX win that should be in the PR description for the changelog.

## What I learned

The "remount-key as change detector" pattern is a common React/Ink anti-pattern for components that live inside a `<Static>`-style append-only region: a key change *does* force a re-render, but in append-only contexts that re-render produces visible duplicate output rather than replacing the prior render. The fix here is the right shape — separate the "detect change" mechanism (pub/sub) from the "re-render" mechanism (explicit refresh API call), so the two can be independently chosen. The bonus win is deleting a per-second polling timer, which matters less for CPU and more for Node's ability to batch-defer wakeups during idle periods.
