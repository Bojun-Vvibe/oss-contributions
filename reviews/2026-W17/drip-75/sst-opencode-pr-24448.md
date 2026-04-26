---
pr: 24448
repo: sst/opencode
sha: 6966a3385677cf27c65a621eff2c78fd3a588dd0
verdict: merge-as-is
date: 2026-04-26
---

# sst/opencode #24448 — fix(app): ignore watchman cookie files

- **URL**: https://github.com/sst/opencode/pull/24448
- **Author**: aakash2330 (Aakash Singh)
- **Head SHA**: 6966a3385677cf27c65a621eff2c78fd3a588dd0
- **Size**: +29/-1 across 4 files (`packages/opencode/src/file/{ignore,watcher}.ts` + two tests)
- **Closes**: #24357

## Scope

Two product changes plus two test additions:

1. `packages/opencode/src/file/ignore.ts:43` — adds `"**/.watchman-cookie-*"` to the OS-junk block of `FILES`, right after `Thumbs.db`. Watchman drops these short-lived sentinel files (`.watchman-cookie-<host>-<pid>-<n>`) into the watch root every time it issues a query, so the project file watcher was firing a `change` event for each one.
2. `packages/opencode/src/file/watcher.ts:142` — when the `.git/HEAD` watcher subscribes to `vcsDir`, it now passes `[...ignore, ...FileIgnore.PATTERNS, ...cfgIgnores]` instead of the previous `ignore` only. Previously the `.git`-scoped watcher was *not* applying the project-level ignore patterns, so a watchman cookie landing inside `.git/` (which Watchman does — it watches the whole tree from the repo root) would still bubble up.
3. Tests in `test/file/ignore.test.ts:10-11` and `test/file/watcher.test.ts:227-251` cover both surfaces: a unit-level `FileIgnore.match(".watchman-cookie-123")` assertion plus an integration-level "write a cookie at project root and inside `.git/`, assert no update fires" using the existing `noUpdate` helper.

## Specific findings

- **Pattern shape is correct.** `**/.watchman-cookie-*` matches `.watchman-cookie-foo` at the root and `nested/.watchman-cookie-foo` in any subdirectory, which is exactly what Watchman produces. The existing tests at `test/file/ignore.test.ts:10-11` cover both. Note that Watchman does not put a leading dot before `watchman-cookie` in some old configurations (it's been `.watchman-cookie-*` since at least 2019), so the dot is safe to anchor on.

- **The `vcsDir` subscribe fix is the more interesting half of this PR.** The diff at `watcher.ts:142` looks like a one-line change, but it's actually a missing-defense fix that's been latent since the `.git/HEAD` watcher was introduced. The original signature only excluded `entry !== "HEAD"` — meaning every other `.git/` mutation (including cookies, `.git/index.lock`, `.git/objects/pack/tmp_*`, `.git/refs/heads/*` rewrites) was firing watcher events even though `FileIgnore.PATTERNS` already lists `**/.git/**` further up. Threading `FileIgnore.PATTERNS` and `cfgIgnores` into the subscribe call is the right fix. Worth a follow-up to confirm there isn't a callsite that *needs* the old "only HEAD ignored" behavior — a quick grep of `subscribe(vcsDir` shows this is the only one.

- **Test setup uses the existing `withWatcher` + `noUpdate` harness** at `test/file/watcher.test.ts:228-249`. The `concurrency: 1` is important here — running the two `noUpdate` checks in parallel would race the `tmpdir` git-init step. The author noticed this; good attention.

- **Pre-existing pattern list is consistent.** Looking at `ignore.ts:31-46`, the OS-junk block already had `.DS_Store` (macOS) and `Thumbs.db` (Windows). Watchman cookies are the same shape of cross-cutting OS junk — slotting it next to those two is the natural home. Not in the "Logs & temp" block below (those are dev-tool outputs the user might actually want to see in some workflows).

- **No new dependency, no config surface added.** The pattern is a hard-coded constant — users on Watchman who somehow want to *see* cookie events have no override. That's fine: nobody legitimately needs to react to a watchman cookie in their project.

- **Author looks like a drive-by but a clean one.** `aakash2330` doesn't appear in this repo's recent commit log. The PR body is short, names the issue it closes, includes the test, and follows the template. No AI-slop signals. Maintainer should land it.

## Risk

Negligible. The ignore-pattern addition is purely additive and the `vcsDir` subscribe-call change is *narrowing* events the watcher reacts to (i.e. no new events get raised). Worst-case regression: a downstream consumer was relying on `.git/index.lock` or `.git/refs/...` events firing through this watcher path, and now they don't. Search for `watcher.subscribe` or `Bus.subscribe("file.")` consumers in `packages/opencode/src/` that filter on a `.git/` prefix — if there are none (likely), this is risk-free.

## Verdict

**merge-as-is** — small, focused, tested at both unit and integration levels, fixes a real user-visible issue (#24357) plus a latent bug nobody had noticed in the `.git/` watcher path. The 4-file diff is exactly the right scope for the change.

## What I learned

When a project ignores a file pattern at one watcher, it should ignore it at *every* watcher rooted in the same tree — otherwise a "global" ignore list silently has holes wherever a sub-watcher was added later with a narrower exclude set. The `subscribe(vcsDir, ignore)` → `subscribe(vcsDir, [...ignore, ...FileIgnore.PATTERNS, ...cfgIgnores])` change is a generalizable lint: anywhere you see `subscribe(path, customIgnores)` without the project-level patterns merged in, that watcher is leaking.
