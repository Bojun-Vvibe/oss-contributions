# sst/opencode #24745 — fix(app): persist project edit metadata

- PR: https://github.com/sst/opencode/pull/24745
- Head SHA: `a3ccb82158336d2867fee36ca3fe197ea8842f57`
- Files changed: 2 (`+8/-6`) — `packages/app/src/context/layout.tsx`, `packages/ui/src/components/avatar.tsx`
- Closes: #24744

## Verdict: merge-after-nits

## Rationale

- **Right diagnosis.** The edit dialog already saves project meta into `globalSync.project.meta` (i.e. `childStore.projectMeta`), but the sidebar reducer at `layout.tsx:386-395` only ever returned `{ ...metadata, ...project }` from the *server* `globalSync.data.project` row. For projects whose server-side metadata row is `id === "global"` (the catch-all bucket for non-explicit projects) or whose row is genuinely absent, the locally edited `name`/`icon`/`commands` were silently dropped — exactly the symptom in #24744. The new condition at `:393` (`!metadata || metadata.id === "global"`) correctly fires for both the "no server row" and "server row exists but is the global bucket" cells.
- **Merge order is intentional and correct.** `:396` `{ ...metadata, ...local, ...project, icon, commands }` puts `local` *between* server `metadata` and the in-flight `project` so live runtime fields (like worktree paths set by `project`) still win, while user-edited static fields like `name` from `local` override stale server name. The `icon`/`commands` values are then explicitly merged at the end as `{ ...metadata?.icon, ...local.icon }` so per-subfield overrides (e.g. user changes only `icon.color` but kept `icon.url` from server) survive — this is the right partial-override shape and mirrors the same pattern landed in drip-134 #24738 for `childStore.icon` per-workspace overrides.
- **Avatar reactivity fix is load-bearing for the UX claim.** `avatar.tsx:33` previously did `const src = split.src` once at component-mount time, so when the user uploaded a new image the `<Show when={src}>` predicate at `:46` never re-evaluated and the fallback initials kept rendering until full remount. The new code inlines `split.src` at the four read sites (`data-has-image`, the two style guards at `:42-43`, and the `<Show when=>`) — Solid's `mergeProps`/destructure semantics make these reactive accessors, so the swap from cached-once to read-each-time is the minimal correct fix. The author's "did this so i can zero it out to test fallback" comment removal (`:26`) is the right cleanup — that test-only gymnastic stops being needed once `split.src` is the canonical accessor.

## Nits / follow-ups

- The merge expression at `layout.tsx:396` now has *six* spread sources combined with two named overrides. Naming the intermediate `const merged = { ...metadata, ...local, ...project }; return { ...merged, icon, commands }` would help future readers see the intent ("merge then override icon/commands explicitly").
- `local.commands` and `local.icon` use the same shallow-merge shape but the truthy-check is on `local?.commands` not on whether commands has any keys — if a user clears all commands the empty `{}` will replace `metadata?.commands` rather than fall through. Probably the intended semantic for "user explicitly cleared," worth a one-line code comment confirming.
- No test was added for the "global metadata row + local override" cell despite this being the exact bug being fixed; a layout-context unit test pinning `result.name === local.projectMeta.name` when `metadata.id === "global"` would lock the regression.
- Avatar's `<Show when={split.src}>` now relies on the truthy-check rejecting empty string — `data-has-image={split.src ? "" : undefined}` at `:33` does the same thing. Both are consistent and correct, no action needed; flagging only because reviewers may double-check.
