# charmbracelet/crush #2808 — fix(ui): prevent duplicate skills from rendering

- URL: https://github.com/charmbracelet/crush/pull/2808
- Head SHA: `52aa09aad1bbb3400f9852cc3befa319805668b3`
- Author: ilgax
- Size: +3 / -0 in `internal/ui/model/skills.go`

## What it does

Adds a 3-line de-duplication guard to `skillStatusItems()` at `skills.go:79-84`: before recording a skill into `stateNames` and appending its status item to the slice, check `if _, exists := stateNames[name]; exists { continue }`. The `disabledSet` short-circuit at `:79` is preserved as-is.

## Observations

- **Symptom is clear from the diff context: the loop iterates a source where the same skill `name` could appear more than once.** The map `stateNames` was already being used as a "seen set" *after* the append, so any duplicate name would (a) overwrite the map entry harmlessly and (b) add a duplicate `skillStatusItem` to the rendered list. The fix flips the order — check-then-mark-then-append — which is the textbook seen-set pattern.
- **The fix is provably correct for the rendering symptom but doesn't address the root cause** (whatever upstream surface is producing duplicate skill names). If the duplicates come from a global+local skill of the same name, silently dropping the second occurrence loses information that the user might want to see (e.g. "global skill X is shadowed by local skill X"). If the duplicates come from a stateful caller adding the same skill twice, the right fix is upstream. The PR description (1-line bug fix) doesn't disambiguate.
- **Order-dependence:** with the new guard, the *first* occurrence wins. The icon and state shown for skill `X` are whichever entry the loop encounters first. If the iteration order over the underlying source is non-deterministic (e.g. map iteration), users may see flickering between two valid representations. Worth confirming the source slice is deterministically ordered.
- The `disabledSet[name]` check at `:79` runs *before* the new guard, which is correct — a disabled skill shouldn't poison the seen-set such that an enabled duplicate also gets dropped.
- No test added. A 5-line table-driven test on `skillStatusItems()` with a duplicate-name fixture would pin the behavior cheaply.

## Verdict: merge-after-nits

Tight, surgical fix for a visible UI bug. Two nits worth a one-line response in the PR thread before merge: (1) is the upstream source of duplicate names known and tracked separately, and (2) is the iteration order of the source deterministic enough that "first wins" is stable across runs? A small regression test would be nice but isn't load-bearing for a 3-line fix.