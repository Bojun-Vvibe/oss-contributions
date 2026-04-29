# sst/opencode #24947 — project `.opencode/` config now overrides global `~/.opencode`

- **PR:** https://github.com/sst/opencode/pull/24947
- **Title:** `fix(opencode): project .opencode/ config now overrides global ~/.opencode`
- **Author:** bainos (Jacopo Binosi)
- **Head SHA:** e94abf0faf681e0ea631084c05a83c3f60a5275b
- **Files changed:** 1 (`packages/opencode/src/config/paths.ts`), +5 / −5
- **Verdict:** `merge-after-nits`
- **Closes:** #19296, #21307

## What it does

Re-orders the array returned by `ConfigPaths.directories()` at
`paths.ts:26-44`. Before: `[Global.Path.config, ...projectWalk(start=cwd, stop=worktree), ...homeWalk(~)]`. After:
`[Global.Path.config, ...homeWalk(~), ...projectWalk(start=cwd, stop=worktree)]`.

The downstream merge order means "later wins" — moving the home walk
*before* the project walk causes project `.opencode/` directories to land
last and therefore override `~/.opencode`. The `unique([...])` wrapper
deduplicates, so when `Global.Path.config === ~/.opencode/config` the
home-walk hit collapses and doesn't double-process. The repro in the PR
body (project `agent/build.md` model = Y, global = X → before fix: X
wins, after: Y wins) confirms the merge-order interpretation.

## Why this is correct

Config search order in OpenCode reads `[Global, ...specific]` and a
later entry overrides earlier entries during merge — i.e. user
expectation is "project beats global". The bug was that the *home* walk
(which produces `~/.opencode`) was appended *after* the project walk, so
home was the last entry and won. Swapping the spreads is the minimum
correct fix and keeps the existing `OPENCODE_DISABLE_PROJECT_CONFIG`
escape hatch intact at `paths.ts:30-37`.

## What's good

- Pure swap, no new state, no behavior change when project config is
  absent (the home walk still runs and `~/.opencode` is still honored).
- Closes two distinct user reports (#19296, #21307) that triangulated
  the same root cause from different symptoms.
- Reproduction recipe in the PR body is concrete and three steps long.
- `unique()` already handles the "user has `~/.opencode` and runs from
  inside `~/`" overlap case, so no extra dedupe logic is needed.

## Nits / risks

- **No test.** This is a precedence rule that the next refactor *will*
  break. A `paths.test.ts` case that asserts `directories()` returns
  `[global, home, ...project]` in that exact order — and a second case
  with `OPENCODE_DISABLE_PROJECT_CONFIG=1` asserting the project walk is
  empty — would lock the contract. Without it, the next contributor who
  "tidies up" this function will re-introduce #19296.
- **Brittleness of "later wins."** The swap relies on whatever consumer
  iterates `directories()` and overlays in order. If even one consumer
  iterates back-to-front or uses `Array.prototype.reverse()`, this fix
  silently flips. Worth grep-ing call sites in `packages/opencode/src/config/`
  to confirm the iteration direction is uniform, and adding a
  `// IMPORTANT: order is precedence — last wins on merge` comment at
  `paths.ts:27` so the invariant survives a future code-style pass.
- **Edge case worth a test:** `cwd === ~` (user runs `opencode` from
  their home directory). Project walk starts at `~` and walks up to
  `worktree`; if `worktree === ~`, the project walk produces `~/.opencode`,
  which then dedupes against the home-walk hit. Confirm `unique()` keeps
  the *first* occurrence (home-walk), not the last (project-walk),
  otherwise the precedence model inverts at `~`.
- The PR body lists `~/.opencode/agent/build.md` as the example, but the
  actual code path searches for `.opencode` directories and a downstream
  loader picks `agent/*.md` from each. A test that exercises the full
  agent-resolution path would verify the merge semantics actually flow
  through to agent definitions, not just the paths array.

## Verdict

`merge-after-nits` — small, correct, well-motivated swap that fixes a
real two-issue regression. Add the order-asserting unit test and the
precedence comment before merging so the next refactor doesn't undo it.
The `cwd === ~` edge case is worth verifying behaviorally (not just
mentally) before stamping.
