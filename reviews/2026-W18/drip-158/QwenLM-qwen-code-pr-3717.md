# QwenLM/qwen-code#3717 — feat(core): add FileReadCache and short-circuit unchanged Reads

- PR: https://github.com/QwenLM/qwen-code/pull/3717
- Head SHA: `ae045d928d9d0d863bd4e7e18258542c0e146e4c`
- Author: wenshao
- Diff: +827/-8 across `packages/core/src/config/config.ts`, `packages/core/src/index.ts`, `packages/core/src/services/fileReadCache.{ts,test.ts}`, `packages/core/src/tools/read-file.{ts,test.ts}`

## What changed

Introduces a session-scoped `FileReadCache` keyed by `(dev, ino)` that tracks Read/Edit/WriteFile operations and lets the `read_file` tool short-circuit a duplicate full-Read with a "file unchanged" placeholder instead of re-emitting the content.

- `packages/core/src/services/fileReadCache.ts` is new (~75 LOC of logic). Cache entries record `lastReadAt`, `lastReadWasFull`, `lastReadCacheable`, `lastWriteAt`. `check(stats)` returns `{state: 'unknown' | 'fresh' | 'stale', entry?}`. The 420-LOC test file pins: same `(dev, ino)` ⇒ fresh; different `mtimeMs` ⇒ stale; different `size` ⇒ stale; rm-and-recreate (new inode, same path) ⇒ unknown.
- `packages/core/src/config/config.ts:551` adds `private readonly fileReadCache = new FileReadCache()` plus a `getFileReadCache()` accessor at line 2501-2506. Each `Config` instance gets its own cache, so a subagent that constructs its own Config starts empty — explicitly documented in the JSDoc.
- `packages/core/src/tools/read-file.ts:130-167` is the integration site. `execute()` now (1) computes `isFullRead = offset === undefined && limit === undefined && pages === undefined`, (2) does `await fs.stat(absPath)` upfront with try/catch fallthrough, (3) consults `cache.check(stats)`, and (4) on `state === 'fresh' && lastReadWasFull && lastReadCacheable && (lastWriteAt === undefined || lastReadAt > lastWriteAt)` returns `unchangedResult(absPath)` — a placeholder ToolResult whose llmContent is `[File <rel> unchanged since last read in this session — content was provided earlier in this conversation. If you modified this file via shell or external tools, re-read with explicit offset/limit to fetch current content.]`.
- After the existing `processSingleFileContent(...)` path, `read-file.ts:186-196` records the result with `cacheable = typeof result.llmContent === 'string' && result.originalLineCount !== undefined` — explicitly *not* caching binary/image/audio/video/PDF/notebook reads, since the model wants the full structured payload re-emitted on those.

## Observations

The `(dev, ino)` keying rather than path keying is the right primitive — `mv`-then-restore, atomic-rename-on-write (`tmp → rename(real)`), and rm-and-recreate all flip the inode and correctly invalidate the cache without false positives. The test at line 131-141 (`rm + recreate scenario: same path, brand-new inode. The cache is keyed by inode, so the new file is genuinely a stranger`) directly pins this.

The `isFullRead` predicate at line 134-137 correctly disqualifies *every* range-scoped Read from the fast-path: `offset`, `limit`, *and* `pages` (PDF page range) all force a fresh fetch. The freshness check `lastReadAt > lastWriteAt` at line 161 is the load-bearing invariant against same-session edits — if the agent Read, then Edit, then Read again, the second Read goes through (good). The test `it('never short-circuits a ranged Read (offset/limit set)', ...)` at line 769 pins the range-vs-full distinction.

Three observations worth raising:

1. **External-mutation blind spot is acknowledged but only in the placeholder text.** The placeholder string at `read-file.ts:269-272` warns the model that "If you modified this file via shell or external tools, re-read with explicit offset/limit." That's a *prompt-engineering* fix to a *correctness* gap — there's no mechanism that detects shell-tool writes (`run_shell_command` invoking `sed -i`, MCP tools writing to the path, another process). The cache's `lastWriteAt` is only updated by the `recordEdit`/`recordWrite` paths inside this codebase. Worth a follow-up: either subscribe the cache to the shell tool's output (best-effort scan for `>`, `tee`, `sed -i`, etc. is brittle but better than nothing) or have `check()` also re-stat the path against the recorded `(mtimeMs, size)` *one more time* before returning fresh. The current `check(stats)` already does this — the caller passes a freshly-`fs.stat`'d Stats — so a shell write that bumps `mtimeMs` *will* mark the entry stale. Good. But a shell write that preserves `mtimeMs` (e.g., `touch -r ref file`, or a same-millisecond write on a coarse-resolution FS) won't. The `size` axis catches most cases, but content-equal writes evade. Acceptable tradeoff, just worth documenting in the `FileReadCache` JSDoc.

2. **`originalLineCount !== undefined` as the cacheability gate** at line 192-193 is a good proxy for "this read produced a clean text frame," but it couples the cache shape to a `processSingleFileContent` implementation detail. A future refactor that defines `originalLineCount` for binary reads (e.g., 0 for empty buffers) would silently start caching binary reads and serving the unchanged placeholder where the model expects the bytes. A more defensive predicate would be `result.contentType === 'text'` or an explicit `result.cacheable` flag set in the producer.

3. **Telemetry hole is explicit and documented** at lines 251-262 — "No `logFileOperation` is emitted on this path: the file_unchanged fast-path bypasses the read pipeline entirely, and the existing `FileOperationEvent` schema has no representation for 'served from cache'. A dedicated cache-hit metric can be added when telemetry needs visibility into the fast-path's effectiveness." That's the right tradeoff for v1, but operationally it means cache-hit rate is invisible until the schema extension lands. Worth filing the follow-up issue at merge time.

The memory-file path at line 209-219 elegantly *reuses* the upfront `stats` (`stats ?? (await fs.stat(absPath))`) so the optimization's extra-stat doesn't compound for memory files — the comment at line 211-213 is exactly right about preserving the fallback shape.

## Verdict

`merge-after-nits`
