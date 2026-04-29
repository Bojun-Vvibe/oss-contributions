# QwenLM/qwen-code PR #3717 — feat(core): add FileReadCache and short-circuit unchanged Reads

- Repo: QwenLM/qwen-code
- PR: https://github.com/QwenLM/qwen-code/pull/3717
- Head SHA: `a97e556b2053083456f00919587a518449143fd3`
- Author: wenshao
- Size: +983 / −8, 7 files

## Context

The Read / Edit / WriteFile tools previously re-emitted the full file
contents on every Read invocation, even when the model had already seen
the same `(dev, ino, mtime, size)`-stable contents earlier in the
conversation. For long sessions with repeated Reads of the same
boilerplate file (large package.json, generated schemas, monorepo root
config), this burns significant context budget. The PR introduces a
session-scoped `FileReadCache` keyed on `(dev, ino)` (real inode, so
hardlinks resolve correctly) with stats validation on every check, and
plumbs it into `Config` so Read can return a `file_unchanged` placeholder
when the model has demonstrably already received the contents in this
session.

## What the diff actually does

Three production wirings + a new service + extensive tests (~420 lines
test, 7 files):

1. `packages/core/src/services/fileReadCache.ts` — new service. Inode
   key is `${dev}:${ino}` (`FileReadCache.inodeKey`), entries hold
   `{realPath, mtimeMs, size, lastReadOptions}`. The check API returns
   a tagged union: `{state: 'unknown'}` (never seen, or rm+recreate ⇒
   different inode), `{state: 'fresh', entry}` (matching dev/ino +
   matching mtime + matching size), `{state: 'stale', entry}` (matching
   dev/ino but mtime or size diverged). The `state: 'unknown'` semantics
   for "same path, different inode" (rm + recreate) is the right call —
   the cache key is the inode, so the new file is genuinely a stranger,
   and Edit/WriteFile callers are forced into "must Read first" rather
   than misled into "stale (you saw an earlier version of this exact
   file)".

2. `packages/core/src/config/config.ts:556-602, 1346-1352, 2515-2530`:
   - Adds `private readonly fileReadCache = new FileReadCache()`
     (one instance per `Config`, which means each subagent's `Config`
     inherently starts fresh — that's the right default).
   - Adds `fileReadCacheDisabled: boolean` config flag with public
     getter `getFileReadCacheDisabled()`. Documented as an "escape
     hatch for sessions where the cache's already-seen-this-content
     assumption is unreliable — e.g. after context compaction or
     transcript transformation".
   - `startNewSession()` at `:1346-1352` calls
     `this.fileReadCache.clear()` with the load-bearing comment "the
     model having seen the prior full read earlier in the *current*
     conversation. Carrying entries across /clear or session resume
     would let a follow-up Read return the placeholder despite the new
     session never having received the file contents." This is the
     correctness invariant the whole feature rides on.

3. `packages/core/src/index.ts:142` exports the new service.

4. Test coverage at `packages/core/src/services/fileReadCache.test.ts`
   (420 lines) hits every transition matrix corner I'd want pinned:
   - `inodeKey` stability and collision-avoidance (different `(dev, ino)`
     ⇒ different keys, including the symmetric pair `(1,2)` vs `(2,1)`).
   - `check` returns `unknown` for never-seen, `fresh` for matching
     stats, `stale` when mtime differs, `stale` when size differs,
     `unknown` (not stale) when only inode differs (the rm+recreate
     case). The "unknown not stale" assertion is the discriminating
     test — it pins the safer semantics against a future contributor
     who would intuitively reach for "stale on any divergence".
   - Entry attachment on `fresh` and `stale` results so callers can
     reuse `entry.realPath` / `entry.lastReadOptions`.

5. Config-level regression test at
   `packages/core/src/config/config.test.ts:317-339` pins that
   `startNewSession()` clears the cache. The comment ("the file-read
   cache backs ReadFile's `file_unchanged` placeholder, whose
   correctness depends on the model having seen the prior read earlier
   in the *current* conversation") encodes the contract for the next
   maintainer.

## Risks and gaps

1. **Inode-based keys break on Windows-style filesystems where `ino`
   isn't stable**. `node:fs.Stats.ino` on NTFS is the file's NTFS
   change-journal USN-derived value — usually stable, but can change
   across some operations (defragment, certain backup-restore paths).
   On `\\wsl$` mounts and various network shares, `ino` may always be
   `0` or wrap. The cache then either silently no-caches everything
   (low cost, regressed perf) or silently treats every file as the
   same file (correctness disaster). Worth either (a) detecting
   `ino === 0` in `recordRead` and refusing to cache, or (b)
   documenting the platform-stability assumption explicitly.

2. **`size` and `mtimeMs` are coarse change detectors**. Two writes
   that flip the same byte count and land in the same millisecond
   (think `sed -i 's/0/1/g' file && sed -i 's/1/0/g' file` in rapid
   succession, or two atomic `rename(2)` swaps that replace a file
   with a same-size mutation faster than the FS clock granularity)
   look identical to the cache. For interactive coding workflows
   that's fine — humans don't edit faster than 1ms. For
   build-system-driven file churn (e.g. a watcher that rewrites a
   generated file mid-conversation), it's a footgun. Adding a
   `mtimeNs` (nanosecond-precision) check where available, or
   optionally a content hash for files under N bytes, would harden
   this without much cost.

3. **The cache is `Config`-scoped, but `Config` lifecycle isn't
   obvious**. The PR comment claims "each subagent that creates its
   own Config inherently starts with an empty cache" — which is
   correct *if* every subagent really constructs its own Config, and
   *not* if a parent shares its Config instance with subagents (which
   would let a subagent's reads silently inherit the parent's cache,
   breaking the "model has seen this content earlier in the
   conversation" invariant for the subagent's separate transcript).
   Worth a one-line invariant comment at the field declaration:
   "MUST NOT share this cache instance across agents/sessions whose
   transcripts are not concatenated."

4. **`fileReadCacheDisabled` is a kill switch but doesn't disable
   `recordRead`**. Looking at the diff, the flag is exposed via
   `getFileReadCacheDisabled()` but I don't see Read/Edit/WriteFile
   call sites in this PR — they're presumably in a follow-up. As long
   as the consumer code checks the flag before *both* reading and
   recording (otherwise the cache fills up while disabled, and
   re-enabling silently turns on stale entries), this is fine.
   Recommend either a getter-only API + a `disable()` that clears, or
   guard `recordRead` to no-op when disabled.

5. **No size cap / eviction policy**. The cache grows unbounded with
   the number of distinct inodes Read in a session. For a session
   that crawls a large monorepo, this is a slow memory leak. An LRU
   bound (e.g. `maxEntries = 1000`) with eviction warning logs would
   prevent runaway growth. Probably fine to defer if the consumer
   side already has reasonable bounds, but worth flagging.

6. **The `FileReadCache` API is missing `invalidate(inodeKey)` /
   `invalidatePath(realPath)`**. When Edit / WriteFile *write*, the
   cache should auto-invalidate (since the file's contents changed
   under the cache's nose). The PR's `recordRead` API is read-side
   only; if a follow-up wires Edit/WriteFile to also call
   `recordRead` after writing the new contents (so the model's
   already-seen state matches the fresh-on-disk state), the cache
   stays consistent — but if the follow-up forgets, the cache will
   serve stale `fresh` results on the next Read after a same-session
   Edit. Recommend exposing `invalidate(inodeKey)` and documenting
   the invariant.

7. **`fileReadCacheDisabled` defaults to `false` (cache active)**,
   which is the right default for the common case but means
   compaction-aware sessions need to know to flip it. A more robust
   path would be: `Config.startNewSession()` already clears the
   cache; have the compaction path *also* call clear (or a new
   `invalidateAll` that nukes entries without the `'unknown'`-vs-
   `'stale'` distinction reset). The flag is then a per-deployment
   override, not a per-event toggle.

## Suggestions

- Detect `stats.ino === 0` in `recordRead` and refuse to cache, OR
  document the inode-stability platform requirement explicitly in the
  `FileReadCache` JSDoc.
- Add a `mtimeNs` (or `mtimeNsBig`) check where Node exposes it, or
  optionally a content hash for small files, so sub-millisecond
  same-size writes don't false-positive as `fresh`.
- Add a one-line invariant comment at the `fileReadCache` field
  declaration: "MUST NOT share this cache instance across agents/
  sessions whose transcripts are not concatenated."
- Either have `FileReadCache.recordRead` no-op when the consumer has
  set `fileReadCacheDisabled`, or ensure the consumer wraps both read
  and write paths in the flag check.
- Add an LRU bound or eviction policy with a warning log when the cap
  is reached.
- Expose `invalidate(inodeKey)` / `invalidatePath(realPath)` so Edit /
  WriteFile callers can keep the cache consistent on writes.
- On compaction, call `fileReadCache.clear()` (or a new
  `invalidateAll`) so the post-compaction transcript can't serve a
  stale `fresh` for content the compacted model state no longer
  references.

## Verdict

**merge-after-nits** — the data shape is right (inode-keyed, stats-
validated, with the discriminating `unknown`-vs-`stale` semantics for
rm+recreate), the lifecycle invariant (clear on `startNewSession`) is
encoded both in code and in a regression test, and the test matrix
covers the corners that matter. The nits are about platform robustness
(Windows/network FS inodes), eviction bounds, and tightening the
write-side of the contract before consumers get wired up.
