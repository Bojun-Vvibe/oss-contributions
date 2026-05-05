# sst/opencode #25778 — fix: refresh config cache after file changes

- URL: https://github.com/sst/opencode/pull/25778
- Head SHA: `3c3145737d57667dbdc8f03ec58b501cf73d1e6a`
- Author: kill74
- Size: +62 / -19 across (config.ts plus tests likely)

## Comments

1. `packages/opencode/src/config/config.ts:289-290` — Adding `files: string[]` and `fingerprints: Record<string, string>` to the cached `State` is the right shape: it captures both the *set* of files that contributed to the merged config and a cheap signature for change detection. Good schema.
2. `packages/opencode/src/config/config.ts:342-350` — `fingerprintFile` uses `${stat.size}:${stat.mtimeMs}` and treats `ENOENT` as the sentinel string `"missing"`. This is a sensible cheap fingerprint — it correctly detects file-creation, file-deletion, and any in-place edit that changes size or mtime. Two nits: (a) `mtimeMs` has sub-millisecond resolution on Linux but truncates on macOS APFS to ~1µs; for a config file that gets rewritten twice in the same millisecond by a tool you'll miss the second change. Consider `${stat.size}:${stat.mtimeNs ?? stat.mtimeMs}`. (b) The `"missing"` sentinel will collide with a real fingerprint of a 7-byte file with mtimeMs=`issing` — impossible in practice but worth a `MISSING_SENTINEL = Symbol.for(...)` if you're being strict.
3. `packages/opencode/src/config/config.ts:352-358` — `fingerprintFiles` runs sequentially. For ~5-10 config files this is fine; if the watch list grows (e.g. all `.opencode/agents/*.json`) consider `Promise.all`. Not a blocker.
4. `packages/opencode/src/config/config.ts:472` — Tracking files via a `Set<string>` and converting to `trackedFiles = Array.from(files)` at the end is the right pattern. The set automatically dedupes the global/local/managed paths that legitimately overlap.
5. `packages/opencode/src/config/config.ts:531-533` — Pre-seeding `files` with `config.json`, `opencode.json`, `opencode.jsonc` from `Global.Path.config` even when those files don't currently exist is the correct call: the fingerprint-comparison loop in `get` needs to detect *creation* of a config file, not just modification of an existing one. Good.
6. `packages/opencode/src/config/config.ts:744-752` — The freshness check inside `get()` is the heart of the change. **Concern**: `Config.get` is now an async-ish call that hits `fsNode.stat` for every tracked file on every read. If `get` is called in a hot loop (per-token, per-tool-call, per-render) this is a real I/O hit on the user's home dir. Two options: (a) add a per-call debounce — only re-fingerprint if `Date.now() - lastCheck > 200ms`; (b) wire to `chokidar`/`fs.watch` and only invalidate on actual fs events. Either is fine; the current "stat-on-every-read" is the simplest correct version and probably acceptable for now, but please add a Log.debug counter so we can spot it if it gets called 10k times a session.
7. `packages/opencode/src/config/config.ts:754-756` — `current.files.some((file) => latest[file] !== current.fingerprints[file])` is correct. Worth extracting to a named `hasAnyFileChanged(current, latest)` for readability.
8. `packages/opencode/src/config/config.ts:781` — Adding `InstanceState.invalidate(state)` to the existing `invalidate` Effect.fn is a legitimate fix for an unrelated stale-cache bug. Worth calling out separately in the PR body — reviewers will otherwise wonder why `invalidate` didn't already invalidate.
9. The big import reordering (lines 1-43) is pure noise from a formatter run. **Please separate** into a prep commit ("chore: sort config imports") so the substantive diff is reviewable. As-is the PR shows ~50 lines of import churn that drowns out the real ~30-line fix.
10. Missing: a test that writes to a config file *after* the first `get()` and asserts the second `get()` returns the new value. The PR claims to fix a cache bug; there should be a regression test that fails on `main`.

## Verdict

`request-changes`

## Reasoning

The fingerprint-based invalidation is the right design and the implementation is mostly clean. Three blockers before merge: (1) split the import-sort noise into its own commit so the real diff is readable; (2) add a test that demonstrates the original cache-staleness bug being fixed; (3) think about the cost of stat-ing N files on every `Config.get()` — at minimum, instrument it. The mtimeMs-resolution and ENOENT-sentinel concerns are nice-to-have.
