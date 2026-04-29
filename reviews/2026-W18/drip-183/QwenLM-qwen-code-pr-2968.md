---
pr: QwenLM/qwen-code#2968
sha: fd9335135a77d0bf7145b5ff7b0e8464f50061a0
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(core): reorder LruCache entries on get() for falsy values

URL: https://github.com/QwenLM/qwen-code/pull/2968
Files: `packages/core/src/utils/LruCache.ts`
Diff: 6+/5-

## Context

`LruCache<K, V>.get` in `packages/core/src/utils/LruCache.ts:16-23`
implemented LRU recency promotion by reading the value, then if-truthy
deleting and re-inserting to push the entry to the end of the JS
`Map` insertion order. The bug: `if (value)` was treating any falsy
stored value (`0`, `''`, `false`, `null`) as a cache miss for the purposes
of recency promotion. So a cache holding `userId → 0` or
`featureFlagName → false` would evict perfectly-live entries on capacity
pressure because `get()` failed to mark them as recently used.

## What's good

- Replaces the value-truthiness check with key-presence: `if (!this.cache.has(key))
  return undefined;` (`LruCache.ts:17-19`) — this is the correct membership
  test for `Map`, distinguishing "absent key" from "present key with
  falsy value."
- The reorder pair `this.cache.delete(key); this.cache.set(key, value);`
  (`LruCache.ts:21-22`) is now unconditional given the key exists, which
  matches the documented LRU contract: every successful `get` is a
  promotion.
- The `as V` cast at `:20` is safe given the prior `.has(key)` guard —
  TypeScript narrows `Map.get`'s `V | undefined` return based on existence,
  but the cast makes the invariant explicit for the reader without
  introducing a runtime check.
- Diff is six lines, single function, no API change. Any caller relying
  on the buggy behaviour (treating falsy `get` results as cache misses)
  was already broken by the more fundamental bug that those entries were
  incorrectly evicted; the fix actually makes the caller's life easier.

## Verdict reasoning

Textbook correctness fix for a classic JS-truthiness-vs-Map-membership bug
in an LRU implementation. The bug class is well-known and the fix shape
matches every other production LRU we've seen (`Map.has` for membership
test, unconditional reorder on hit). No tests added in the diff which is
a minor miss, but the fix itself is obviously correct from inspection
and the cache class is small enough that the reviewer can verify by
reading the whole file. Land it.
