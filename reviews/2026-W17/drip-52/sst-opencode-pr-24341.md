---
pr: 24341
repo: sst/opencode
sha: 28d0790c26b394f9dada9128f93ed1ec21c8eeae
verdict: merge-as-is
date: 2026-04-26
---

# sst/opencode#24341 — fix(app): normalize watcher paths

- **URL**: https://github.com/sst/opencode/pull/24341
- **Author**: pascalandr
- **Files**: `packages/app/src/context/file/watcher.ts`,
  `packages/app/src/context/file/watcher.test.ts` (+72/-3)

## Summary

Two real bugs in `invalidateFromWatcher`:

1. On Windows, the file watcher emits backslash-separated paths
   (`src\open.ts`) but the rest of the app keys open files by
   forward slash. Result: a save in VS Code on Windows didn't
   trigger the in-memory file refresh because `isOpen("src\open.ts")`
   missed against keys like `"src/open.ts"`.
2. For `add` / `unlink` events on a deeply nested path, the
   refresh logic only looked at the *immediate* parent directory.
   If the immediate parent wasn't currently loaded in the tree
   (common — directories are lazy-loaded), the event was dropped
   even when an ancestor *was* loaded and would have wanted to
   know about the new entry.

## Reviewable points

- `packages/app/src/context/file/watcher.ts:18-20` — `toWatcherKey`
  is the one-liner normalizer:
  `path.replace(/\\/g, "/")`. Applied at line 41 *after*
  `ops.normalize`, so any provider-specific normalization runs
  first and the slash flip happens last. Right ordering — this
  way `ops.normalize` can still receive its native path format on
  platforms that need it.

- `watcher.ts:22-30` — `nearestLoadedParent` walks upward from
  the file's immediate parent until it finds a loaded directory
  or hits root. Returns `undefined` if no ancestor is loaded. The
  `while (true)` + `parts.pop()` after the `length === 0` check
  is correct: when `parts` is `["src", "nested", "deep"]` (file
  was `src/nested/deep/new.ts`), iterations check
  `"src/nested/deep"`, `"src/nested"`, `"src"`, `""` and finally
  return undefined. The early-return on line 26 means root
  (`""`) is *also* considered a candidate for `isDirLoaded`,
  which matches the existing convention elsewhere in this file
  (root is keyed as empty string).

- `watcher.ts:64-65` — call site swap: the old
  `path.split("/").slice(0, -1).join("/")` is replaced with
  `nearestLoadedParent(path, ops)` and the guard becomes
  `parent === undefined`. Note: `parent` can be the empty string
  when root is loaded — that's *truthy as a directory key* in
  this module, so the explicit `=== undefined` is the right check
  vs. a plain falsy guard.

- The two new tests (`watcher.test.ts:60-95`) cover exactly the
  two reported scenarios:
  1. `"src\open.ts"` → loads `"src/open.ts"` (Windows separator
     normalization).
  2. `"src/nested/deep/new.ts"` add event with only `"src"`
     loaded → refreshes `"src"` (ancestor walk).

## Risks

- Symlinks / paths that *legitimately* contain a backslash on
  Linux (rare but possible — `\` is a valid filename character on
  POSIX) will be rewritten. Trade-off the project clearly accepts
  given watcher paths in practice never contain literal
  backslashes on Linux.
- Walking to root on every add/unlink is O(depth) but depth is
  usually <10; not a hot-path concern.

## Verdict

`merge-as-is`. Two small, well-scoped fixes; clean test coverage;
the predicate change from "immediate parent loaded" to "any
ancestor loaded" matches user expectations for lazy-loaded trees.

## What I learned

Lazy-loaded tree views need watcher invalidation to walk *up* to
the nearest materialized node, not just check the immediate
parent. Otherwise you silently drop add events for subtrees that
the user has scrolled away from but whose ancestor is still
visible.
