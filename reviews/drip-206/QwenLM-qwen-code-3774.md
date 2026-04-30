# QwenLM/qwen-code #3774 — feat(core): enforce prior read before Edit / WriteFile mutates a file

- Head SHA: `9b0bdf8d9be9c8497c09b2b0b8ed142782b71e2d`
- Files: `packages/core/src/tools/edit.test.ts`, `packages/core/src/tools/edit.ts`,
  `packages/core/src/tools/read-file.test.ts`, `packages/core/src/tools/read-file.ts`,
  `packages/core/src/tools/tool-error.ts`, `packages/core/src/tools/write-file.test.ts`,
  `packages/core/src/tools/write-file.ts`
- Size: +405 / -2

## What it does

Closes a real and important agent-correctness gap that the session-scoped
`FileReadCache` from #3717 left open. Previously, the model could emit
`EditTool({old_string: "...", new_string: "..."})` against any file path
without ever having read it; if the imagined `old_string` happened to occur
in the on-disk content, the edit succeeded — even though the model was
operating on guesses. The previous safeguard ("0 occurrences" failure) only
caught the imagined-string-doesn't-match case, not the more dangerous
plausible-but-stale match.

This PR makes prior read mandatory for any mutation of a *pre-existing* file
by introducing two new error codes in `tool-error.ts` —
`EDIT_REQUIRES_PRIOR_READ` and `FILE_CHANGED_SINCE_READ` — and a
`requirePriorRead()` private method on `EditToolInvocation` at
`edit.ts:546-595`. The check runs before the write path at `edit.ts:390-401`,
gated by `!editData.isNewFile && !this.config.getFileReadCacheDisabled()`,
which preserves both legitimate exemptions:
new-file creation has nothing to read first, and the existing
`fileReadCacheDisabled` escape hatch falls back to pre-cache byte-for-byte
behavior.

## What works

- **Three-state cache check.** `requirePriorRead()` calls
  `getFileReadCache().check(stats)` and branches on `'fresh' | 'unknown' |
  'stale'` (edit.ts:564-594), giving distinct error messages: "has not been
  read in this session" vs "has been modified since you last read it". Both
  messages name the `read_file` tool the model should use, which is the
  right level of LLM-friendly remediation.
- **Stat failure is non-blocking** (edit.ts:558-561). If the file vanished
  between `fileExists` and the stat call, returning a synthetic "you must
  read first" would be misleading — the existing write path will surface a
  richer error. This is the correct call.
- **Auto-memory recording fix** at `read-file.ts:147-160`. Auto-memory files
  must skip the `<system-reminder>` fast-path (they own a per-read freshness
  reminder) but they MUST still be *recorded* in the cache so a follow-up
  Edit isn't rejected. The PR splits the single `cacheEnabled` boolean into
  `cacheEnabled` (governs `recordRead`) and `useFastPath` (governs the
  unchanged-placeholder lookup). The test at `read-file.test.ts:712-746`
  locks this distinction with a dedicated assertion:
  `expect(fileReadCache.check(fs.statSync(memFile)).state).toBe('fresh')`.
- **Edit-chain semantics preserved.** The `'allows a chain of edits without
  re-reading between them'` test at `edit.test.ts:1010-1029` verifies that
  `recordWrite` after the first Edit re-stamps the cache fingerprint so the
  second Edit's `check(stats)` returns `'fresh'`. Without this, the
  enforcement would force a Read between every consecutive Edit on the same
  file — unworkable.
- **Stale detection test** at `edit.test.ts:953-980` uses `fs.utimesSync`
  with a 60-second future date to defeat coarse-resolution filesystems
  where a same-second mtime would otherwise look unchanged. Solid.
- **Disabled-cache bypass** at `edit.test.ts:1031-1048` confirms that when
  `getFileReadCacheDisabled()` returns true, an unread file *still* edits
  successfully — the operator escape hatch works.

## Concerns

1. **WriteFile parity.** The PR title and motivation cover both
   `EditTool` and `WriteFileTool`, and the file list includes
   `write-file.ts`/`write-file.test.ts`, but the diff excerpt I have only
   shows the EditTool side. Worth confirming the same `requirePriorRead()`
   gate is wired into WriteFile with parallel test coverage and the
   same `isNewFile` exemption — WriteFile's "create or overwrite"
   semantics make the `isNewFile` distinction subtler than Edit's.
2. **Race window.** `requirePriorRead()` stats the file (line 559), then
   the write path stats again. An external mutation in between could let
   a `'fresh'` check be followed by a write against drifted bytes. This
   is fundamentally unsolvable without holding an fd, and the existing
   filesystem race window is unchanged, but worth a one-line comment.
3. **Cache eviction visibility.** If `FileReadCache` ever evicts entries
   (LRU, memory pressure), a long-running session could legitimately
   have read a file 5,000 messages ago and now be told to re-read. The
   error message doesn't distinguish "never read" from "evicted from
   cache", which would help debugging in the field.

The new-file-exempt logic at `edit.test.ts:982-1000` confirms that
`old_string === ''` on a non-existent path bypasses enforcement and lands
the file with the expected content. That's the correct exemption.

Verdict: merge-after-nits
