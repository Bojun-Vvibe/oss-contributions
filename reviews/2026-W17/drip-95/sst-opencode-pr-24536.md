# sst/opencode #24536 — fix(app): hide synthetic global project on home

- Author: arnavpisces
- Head SHA: `383229a549d804e6cba7b3ecf24638776014c41b`
- +2 / −1 in `packages/app/src/pages/home.tsx`
- PR link: <https://github.com/sst/opencode/pull/24536>

## Specifics

- The fix at `home.tsx:28` adds a single `.filter()` to the `recent` projects pipeline: `.filter((project) => project.id !== "global" || project.worktree !== "/")`. Read carefully — this is "drop the project if it is BOTH `id === "global"` AND `worktree === "/"`", i.e. the synthetic placeholder used when the user has no real projects yet. Any real project with a different worktree (or with a different id) survives.
- Companion change at `home.tsx:90`: `<Match when={sync.data.project.length > 0}>` becomes `<Match when={recent().length > 0}>`. Without this, the previous condition `length > 0` would be `true` even when the user has *only* the synthetic global project — the recent-projects section header would render with an empty list underneath. Switching the `<Match>` to read off the already-filtered `recent()` memo means the section disappears entirely when nothing real is left, which is the right UX.
- The boolean expression is unconditionally cheap to evaluate (string comparisons inside an existing `.filter`/`.sort`/`.slice` chain), so no perf concern.

## Concerns

- The `id === "global" && worktree === "/"` predicate hard-codes the synthetic-project sentinel in the view layer. The same shape is presumably defined where the synthetic project is created (likely in `sync.data.project` materialization) — a named constant or a `project.synthetic === true` flag would be more robust to a future rename of the `global` id. This is a one-line fix scoped to home page; centralization can come later.
- No test added. The fix is small enough that a snapshot test or a `recent()` unit test on the memo is overkill, but at least one comment marker (`// synthetic placeholder when no real projects exist`) inside the `.filter()` would make the next reader's grep faster.
- The PR doesn't update the analogous `Match` at any sibling page (e.g. project-picker, sidebar). If the synthetic project leaks into other "recent projects" surfaces, this fix only patches the home page.

## Verdict

`merge-after-nits` — correct, minimal, and the dual change (filter + Match-condition swap to `recent()`) shows the author understood why the empty-section bug existed. Nits: name the sentinel, and confirm no other view renders the synthetic global project the same way.
