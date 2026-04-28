# google-gemini/gemini-cli#26139 — fix(cli): fix bugs from stale closures in FooterConfigDialog

- **Repo:** [google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli)
- **PR:** [#26139](https://github.com/google-gemini/gemini-cli/pull/26139)
- **Head SHA:** `c5782842215daf5968585d62d45775127d45151f`
- **Size:** +180 / -37 (4 files: `FooterConfigDialog.{tsx,test.tsx}`, `useSelectionList.{ts,test.tsx}`)
- **State:** OPEN
- **Fixes:** #26138

## Context

Two user-visible bugs in the `/footer` config dialog, both rooted in the
same React anti-pattern:

1. **Reset to Default needs two presses** — the first press silently
   no-ops, second press works. Caused by a stale `settings.merged`
   closure: the reducer recomputed defaults from a stale settings
   snapshot that already reflected the user's previous changes, so
   "reset" was a no-op.
2. **Active highlight points at old index after reset** — after
   resetting, the focused item's position pointer was still keyed off
   the pre-reset list.

## Design analysis

### Reducer state shape change

Before: `FooterConfigState = { orderedIds, selectedIds }` plus a generic
`SET_STATE` action with a partial-state payload. `showLabels` lived
outside the reducer — it was read from `settings.merged` at handler
invocation time, which is exactly where stale closures bite.

After (`FooterConfigDialog.tsx:42-86`):

```ts
interface FooterConfigState {
  orderedIds: string[]
  selectedIds: Set<string>
  showLabels: boolean   // <-- now in reducer state
}

type FooterConfigAction =
  | { type: 'MOVE_ITEM'; id: string; direction: number }
  | { type: 'TOGGLE_ITEM'; id: string }
  | { type: 'TOGGLE_LABELS' }
  | { type: 'RESET'; payload: FooterConfigState }
```

Two correct moves here:

1. **`SET_STATE` (partial replace) → `RESET` (full replace)**. Partial
   replaces are how you accumulate stale fields by accident — caller
   forgets to include a field, the old value persists. The new `RESET`
   action takes a full `FooterConfigState`, so the reducer is forced to
   produce a complete snapshot every reset.
2. **`showLabels` migrated into reducer state**. Previously the toggle
   wrote directly to `setSetting` (`:165` old code). Now it dispatches
   `TOGGLE_LABELS` and the persistence happens at confirm time
   (`:163-168`). Removes the read-from-merged-settings race.

### Reset handler — the actual fix

`FooterConfigDialog.tsx:174-198` rebuilds the default state from a
synthetic settings object:

```ts
const handleResetToDefaults = useCallback(() => {
  const defaultFooterSettings = {
    ...settings.merged,
    ui: {
      ...settings.merged.ui,
      footer: {
        ...settings.merged.ui.footer,
        items: undefined,
      },
    },
  };
  const defaultState = resolveFooterState(defaultFooterSettings);
  dispatch({
    type: 'RESET',
    payload: {
      ...defaultState,
      showLabels: getDefaultValue('ui.footer.showLabels') !== false,
    },
  });
  setFocusKey(defaultState.orderedIds[0]);
}, [settings.merged]);
```

Key insight: instead of *writing* `items = undefined` to the settings
store and then re-reading the resolved defaults (the old approach,
which required two presses because the write hadn't propagated yet),
this **synthesizes** the "items unset" settings shape locally and runs
`resolveFooterState` against it. Pure function, no async store update,
single dispatch. First-press correctness is now structural rather than
race-dependent.

`setFocusKey(defaultState.orderedIds[0])` at `:196` fixes the second
bug — the focus pointer is recomputed from the new ordered list, not
preserved from the stale list.

### Test coverage

Two new tests at `FooterConfigDialog.test.tsx:264-336`:

```ts
it('reset to default works on first press (no double-press needed)', ...)
it('active index moves to first item after reset', ...)
```

Both drive the dialog through `stdin.write` and assert the rendered
frame contains the post-reset state. The first test pins the
specific bug (workspace unchecked → reset → workspace checked); the
second pins the focus indicator (`> [✓] workspace`). Right shape for a
TUI regression test — uses the public input/output surface rather than
inspecting reducer internals, so the test is robust to internal
refactors.

The `useSelectionList` test additions (`:27` adds, paired with hook
changes `:24/-19`) suggest the same stale-closure pattern was also
present in the selection-list hook itself. Worth reading those changes
to confirm they're scoped to the same fix.

## Risks / nits

1. **`getDefaultValue('ui.footer.showLabels') !== false` is awkward.**
   The `!== false` idiom is "default to true unless explicitly false,"
   which is correct semantics, but it's repeated in multiple places
   (`:103`, `:194`, `:166`). A named helper `defaultShowLabels()` would
   make the intent grep-able and put the truthy-default convention in
   one place.

2. **`useMemo` on `initialState` (`:99-105`) re-runs whenever
   `settings.merged` changes.** That's correct for initial mount, but
   `useReducer`'s lazy initializer only runs once — passing a memoized
   object whose identity changes won't re-initialize the reducer. So
   the `useMemo` is effectively dead code after first render. Either
   drop the `useMemo` (just call `resolveFooterState` inline in the
   initializer) or document why the recomputation is intended.

3. **`RESET` with full payload removes the symmetry with `MOVE_ITEM`/
   `TOGGLE_ITEM`** (which take partial inputs and merge). That's fine
   because `RESET` is semantically different (it's a full overwrite),
   but a reducer comment naming the policy ("RESET fully replaces
   state; other actions merge") would help future maintainers not
   accidentally make `RESET` partial again.

4. **Test brittleness on visual indicators.** The new tests assert
   exact strings like `'> [✓] workspace'` — if the dialog ever changes
   from `>` to `▶` or from `[✓]` to `☑`, both tests break for cosmetic
   reasons. Acceptable trade-off (the tests pin user-visible state) but
   worth a constant for the marker chars to make a future renderer
   change a one-line test update.

5. **No test for the settings-write path on confirm.** The fix moved
   `showLabels` persistence from "write on toggle" to "write on
   confirm" (`:163-168`). That changes the behavior visible to anyone
   else reading from `settings.merged.ui.footer.showLabels` between
   toggle and confirm. Worth one test that toggles labels, exits
   without confirming, and asserts the setting is unchanged.

## Verdict

**merge-after-nits.** Correct diagnosis (stale closure in handler that
read from settings store before its own write propagated), correct fix
(synthesize the default-state input locally, single dispatch), good
test coverage on the user-visible bugs. Pre-merge: address nit 2
(dead `useMemo`) — it's a real footgun for the next maintainer who
sees the memo and assumes it does something. Nits 1, 3, 4, 5 are
follow-up cleanup.

## What I learned

- "Write to store, then re-read" inside a single React callback is a
  classic stale-closure trap when the store update is async. The fix
  is almost always "don't round-trip through the store — compute the
  desired state locally and dispatch it directly."
- Migrating a piece of state from "live-read from external source" to
  "owned by reducer" is the right move when the bug shape is "the
  external read sees a stale value." Once it's in the reducer, all
  reads come from the same atomic snapshot per render.
- `SET_STATE` partial-payload reducer actions are an anti-pattern.
  They look flexible but invite "I forgot to include this field" bugs
  exactly like the one this PR fixes. Named actions with full payloads
  surface the missing fields at the type-checker.
