# BerriAI/litellm PR #26601 — fix infinite re-render loop in VirtualKeysTable

- **PR**: https://github.com/BerriAI/litellm/pull/26601
- **Head SHA**: (per `gh pr view`)
- **Size**: small UI fix + 2 regression tests
- **Files**: `ui/litellm-dashboard/src/components/VirtualKeysPage/VirtualKeysTable.tsx`, `ui/litellm-dashboard/src/components/key_team_helpers/filter_logic.test.tsx`

## Summary

Classic React identity-stability bug: `VirtualKeysTable` was passing `keys?.keys || []` directly to the `useFilterLogic` hook. When `keys?.keys` is `undefined` or `null`, `|| []` synthesizes a brand-new array literal on every render. The hook's internal `useEffect([keys, filters])` then sees a fresh reference each render, calls `setFilteredKeys`, which re-renders the consumer, which produces another `[]`, ad infinitum. The fix wraps the fallback in `useMemo`: `const keysList = useMemo(() => keys?.keys ?? [], [keys?.keys])`.

## Verdict: `merge-as-is`

Correct diagnosis, minimal fix, and the test is exactly the right shape — it asserts the *non*-loop behavior, not just "filteredKeys equals expected". I'd merge as-is.

## Specific references

- `ui/litellm-dashboard/src/components/VirtualKeysPage/VirtualKeysTable.tsx:97` — the new `const keysList = useMemo(() => keys?.keys ?? [], [keys?.keys])` is the entire fix. The `useMemo` deps array is `[keys?.keys]`, so when `keys?.keys` is `undefined` (or a stable reference), the memo returns the same `[]` instance across renders. The downstream `useFilterLogic({ keys: keysList, ... })` then sees a stable reference, and its `useEffect([keys, filters])` no longer fires on every render. Note the switch from `||` to `??`: this is also a correctness improvement — `||` would have replaced any falsy value (including a deliberately-empty array? no, `[] || []` returns the first since `[]` is truthy in JS — so `||` was actually fine here, but `??` is more precise and matches modern conventions).
- `ui/litellm-dashboard/src/components/VirtualKeysPage/VirtualKeysTable.tsx:101-105` — the `useFilterLogic({ keys: keysList, ... })` call site is unchanged in shape; only the value passed in is now stable. This is the right level of intervention — the hook itself doesn't need to change, because the contract is "pass me a stable array". Patching the hook to internally tolerate identity-unstable inputs would have masked the bug for future callers.
- `ui/litellm-dashboard/src/components/key_team_helpers/filter_logic.test.tsx:35-57` — `test("should not enter an infinite render loop when keys prop is re-rendered with a new empty-array reference")` is the regression test. It uses `renderHook` + three explicit `rerender({ keys: [] })` calls (each producing a fresh `[]` literal) and then asserts `result.current.filteredKeys).toEqual([])`. The genius of the test is the *implicit* assertion: if the hook were looping, the test would hang or hit React's `Maximum update depth exceeded` and fail with a different error. The test name calls this out: "should not enter an infinite render loop" — a pure liveness assertion.
- `ui/litellm-dashboard/src/components/key_team_helpers/filter_logic.test.tsx:59-73` — the second test, `test("should update filteredKeys when keys prop changes from empty to populated")`, pins the *positive* case: the memo must not be too aggressive and prevent legitimate updates. Together with the first test, this brackets the behavior on both sides.

## Nits

None. The comment at line 95-96 (`// Stable reference: \`|| []\` creates a new array literal on every render, which / // causes the useEffect in useFilterLogic to fire on every render → infinite loop.`) is exactly the right level of detail to leave for the next reader. It explains *why* the `useMemo` is here, not just *what* it does.

## What I learned

The "fresh empty array" pattern (`thing || []`, `thing ?? []`, `thing.foo?.bar ?? []`) is one of the most common identity-stability bugs in React-with-hooks code, and it's especially insidious because:
1. It often only loops in production-like conditions where parent re-renders happen frequently (e.g., a websocket pushing updates, a parent keeping query state in a hook).
2. The synthetic empty array passes all "shape" assertions in tests — `toEqual([])` works fine on each new instance.
3. The infinite loop is asymmetric: the *consumer* has the bug but the *hook* gets blamed because that's where the `useEffect` fires.

The right test pattern, demonstrated here, is the *liveness* test: render with three identity-distinct empty arrays in sequence and assert that the hook returns. If it were looping, React's reconciler would either hang the test or throw `Maximum update depth exceeded`. This is much more reliable than asserting render counts, which can drift with React versions.
