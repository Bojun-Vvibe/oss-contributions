# sst/opencode #24738 — fix(app): preserve per-workspace icon override from localStorage

- PR: https://github.com/sst/opencode/pull/24738
- Head SHA: `cc8441d9cbdb0da1bbb6c2572042ac4d03c5011d`
- Files changed: 1 (`+8/-1`) — `packages/app/src/context/layout.tsx`

## Verdict: merge-as-is

## Rationale

- **Surgical regression fix with a named blamed commit.** PR body identifies commit `6002500bc` as the change that dropped the `childStore.icon` fallback when enriching project metadata — i.e., `{ ...metadata, ...project }` clobbered any per-workspace icon override stored in `localStorage` because `project.icon` (DB-derived) won the spread. The fix at `layout.tsx:391-401` reads `childStore.icon` after the spread and conditionally re-applies it to `base.icon.override`, preserving the existing path when no override exists.
- **Correct merge shape.** The new code does `{ ...base.icon, override: childStore.icon }` rather than `{ override: childStore.icon }` so other `icon` subfields (presumably `default`/`source`/etc.) survive the override application — that's the right additive merge for a partial override field. Returning `base` unchanged in the `if`-not-taken branch preserves the no-override-set fast path with zero allocation overhead.
- **Symptom is exactly what the diff predicts.** "Different subdirectories of the same git repo share the same icon" is the expected user-visible behavior when the DB-stored icon (keyed by `worktree`) wins over the per-workspace `localStorage` override (keyed by something narrower like child-path) — the `globalSync.data.project.find((x) => x.worktree === project.worktree)` lookup at `:389-390` confirms the DB key is the worktree, so two child workspaces in the same worktree share a row, and the override is the only way to differentiate them.
- **Zero blast radius.** Single-file, 8-line additive change inside one already-existing `enrich` closure at `:386-394`. No new imports, no API surface change, no test touched (none existed for this path). The `if (childStore.icon)` truthy check correctly handles the empty-string and `undefined` cases without falling into the merge branch.

## Nits / follow-ups

- A one-cell test pinning "two children of the same worktree with different `localStorage` overrides resolve to different `icon.override` values" would prevent the same regression class from recurring on the next refactor of this `enrich` closure.
- Worth a one-line code comment naming commit `6002500bc` as the regression source so future archaeology doesn't have to `git blame` the spread.
