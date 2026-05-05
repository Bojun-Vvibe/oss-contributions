# BerriAI/litellm PR #27238 — fix(ui): drop duplicate /v2/team/list fetch racing OldTeams' paginated fetch

- URL: https://github.com/BerriAI/litellm/pull/27238
- Head SHA: `2c801febffdea18299790e3db182d19e80f1a69a`
- Size: +4 / -3 (single React component)

## Summary

Surgical state-isolation fix in the legacy `OldTeams.tsx` dashboard component: introduces a separate local state slice `paginatedTeams` so two different fetch flows (the parent-injected `teams` prop and the component's own paginated `/v2/team/list` call) no longer mutate the same store. Previously both paths were calling `setTeams(...)`, which means whichever resolved last won — producing the visible bug of "duplicate fetch racing the paginated fetch."

## Specific findings

- `ui/litellm-dashboard/src/components/OldTeams.tsx:200` — new state field `const [paginatedTeams, setPaginatedTeams] = useState<Team[] | null>(null);`. Initial `null` (vs `[]`) lets `displayTeams` distinguish "haven't fetched yet" from "fetched, got zero." Correct typing.
- `OldTeams.tsx:244` — first paginated fetch site swaps `setTeams(response.teams ?? [])` → `setPaginatedTeams(response.teams ?? [])`. Preserves the `?? []` defensive default.
- `OldTeams.tsx:663` — second paginated fetch site (the filter-change handler) gets the same swap. Both call sites now go to the local slice; the parent-prop `teams` is never mutated by this component.
- `OldTeams.tsx:867` — `displayTeams` memo flipped from `teams ?? []` → `paginatedTeams ?? []`. **Subtle correctness question:** what about the *initial* render before any paginated fetch has resolved? The component used to show `teams` (from props) and now shows `[]` until the first `/v2/team/list` response lands. Whether that's a regression or an improvement depends on whether the parent ever passes a meaningful `teams` prop or if it was always going to be overwritten. Looking at the diff, the component still receives `teams` via the destructure (not visible in this hunk but implied by the `setTeams(...)` calls being mutations on a parent-passed setter), so the parent likely populates `teams` initially as a quick paint, then the paginated fetch refines it. After this change, that quick-paint is gone and users will see an empty list / loading spinner during the first fetch. Worth a maintainer-side UX confirm before merge — the PR title implies the duplicate-fetch race was a real bug, so trading off the quick-paint to fix it may be the right call, but a `displayTeams = useMemo(() => paginatedTeams ?? teams ?? [], [paginatedTeams, teams])` would preserve both behaviors.
- The `setTeams(...)` setter (and presumably the `teams` prop) is left untouched in the destructure. So the shared store still exists and other consumers (the surrounding `Teams` component? a sibling?) that rely on `setTeams` propagation are unaffected by this PR. The fix is correctly scoped to *only* the display path inside `OldTeams`.

## Notes

- No tests in the diff. This component is in `ui/litellm-dashboard/src/components/` which (in litellm) historically has thin React Testing Library coverage; not a hard blocker but a `displayTeams reflects paginatedTeams once /v2/team/list resolves` test would pin the fix.
- PR title says "drop duplicate /v2/team/list fetch" but the fix doesn't actually drop a fetch — it isolates the *target* of the second fetch's setter so the two fetches no longer fight over the same state slice. Title is misleading; suggest renaming to "fix(ui): isolate paginated /v2/team/list result from parent-passed teams prop."
- Consider following up by deleting the parent-passed `teams` prop entirely if no other consumer reads it through this component, since the fix has effectively orphaned it for display purposes.

## Verdict

`merge-after-nits`
