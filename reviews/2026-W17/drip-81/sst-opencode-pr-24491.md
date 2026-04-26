# sst/opencode PR #24491 — fix(app): sync session sidebar loading and title generation across project context

- **PR:** https://github.com/sst/opencode/pull/24491
- **Author:** theprismdata
- **Head SHA:** `c035c9f82537`
- **Stats:** +51 / -23 across 6 files
- **Verdict:** `merge-after-nits`

## What this changes

Fixes a long-running paper-cut where the sidebar in the desktop app showed stale or empty session lists when the user navigated between projects via deep-link / cwd change without an explicit project pick. The fix has four moving parts that together close the gap:

1. **Eager session-list refetch on session create** (`packages/app/src/components/prompt-input/submit.ts:377`) — after `seed(sessionDirectory, created)`, an explicit `void globalSync.project.loadSessions(sessionDirectory)` makes the new session show up in the sidebar of the originating project even if the project context isn't currently active.
2. **Cross-directory store fan-out** in `sync.tsx:472-488` — the upsert that previously only wrote to the local `setStore` is now extracted to an `upsertSession(setter)` helper and replayed against `globalSync.child(data.directory)` whenever `data.directory !== directory`. This is the actual core fix: a session whose canonical `data.directory` differs from the directory it was streamed into now lands in *both* projects' stores.
3. **Bootstrap-on-resolve** in `pages/layout.tsx:569` — the `globalSync.child(directory, { bootstrap: true })` flip plus the new `createEffect(on(() => [route().slug, params.id, currentDir()], …))` at `:1842-1852` ensures the project's session list is loaded the moment a route mentions a directory.
4. **`panelProject` memoization** at `:2343-2353` — when no project is active but `currentDir()` resolves to a known root, the sidebar now renders for that project instead of the literal first project in the list.

The supporting `helpers.ts:31-36` rewrite of `isRootVisibleSession` to take the whole `SessionStore` (so it can match by `projectID` first and fall back to `workspaceKey`) is the right shape — it makes the visibility predicate explicit about *why* a session belongs to a sidebar.

## Things to fix before merging

The drive-by `packages/opencode/src/session/prompt.ts:170-173` deletion of the `if (input.history.filter(real).length !== 1) return` guard, and the `packages/opencode/src/session/session.ts:753-757` removal of the directory equality filter under `Flag.OPENCODE_EXPERIMENTAL_WORKSPACES`, are both *behavior* changes that have nothing to do with the sidebar bug in the PR title. The session.ts change in particular makes session listing project-scoped instead of directory-scoped *unconditionally* — that's likely the right long-term direction (and the comment "Project scoping is already enforced via project_id above" is correct for the post-experimental-flag world), but it deserves either its own PR or at minimum a callout in the description so reviewers know to think about pre-`projectID` rows where `project_id` could be NULL. Same for the prompt.ts gate removal: the previous code only generated a title on the first real user message; deleting that check means titles can now be regenerated mid-session, which may or may not be intended.

The `void` on `globalSync.project.loadSessions(sessionDirectory)` at `submit.ts:377` discards both the promise and any error — fine for a fire-and-forget refresh, but the sibling `loadSessions` call inside the new `createEffect` at `layout.tsx:1846` is also fired without await/error handling, so a transient sync failure during route navigation will silently leave the sidebar stale (the same bug class this PR is trying to fix). Worth a short `.catch(log)` or routing it through whatever error sink the rest of `sync.tsx` uses. Finally, the `panelProject` memo's last-resort `return projects()[0]` keeps the pre-fix behavior for the empty-`currentDir` path, which is fine — but worth confirming with a quick test that mobile mode doesn't render `<SidebarPanel project={panelProject} mobile />` against a project the user explicitly de-selected.

## Bottom line

Real bug, correctly diagnosed, fix is in the right layers. Split out (or call out) the prompt.ts + session.ts behavior changes, add error sinks to the two new `loadSessions` calls, and this is mergeable.
