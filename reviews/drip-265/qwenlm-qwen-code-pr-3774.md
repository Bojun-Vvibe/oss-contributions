# QwenLM/qwen-code #3774 — feat(core): enforce prior read before Edit / WriteFile mutates a file

- **Repo:** QwenLM/qwen-code
- **PR:** #3774
- **Head SHA:** `303b6b7d4ac7706d78fc3f4e40c6bef222fa7c9d`
- **Verdict:** merge-after-nits

## Summary

Closes a real correctness gap: before this PR, Edit could succeed on
a file the model never Read in this session, as long as `old_string`
happened to occur in the bytes on disk. That allowed plausible-but-stale
edits — model working from imagination rather than evidence.

PR turns the session-scoped `FileReadCache` from PR #3717 into an
enforcement primitive. New error codes:

- `EDIT_REQUIRES_PRIOR_READ` — file has no cache entry.
- `FILE_CHANGED_SINCE_READ` — entry exists but mtime/size drifted.

## Specific notes

- **`fileReadCache.ts:118-142`:** `recordWrite` now seeds
  `lastReadAt`/`lastReadWasFull`/`lastReadCacheable` when the entry
  is new. Correct: a model that just authored bytes via WriteFile has
  effectively "seen" them, and without this seeding a `WriteFile →
  Edit → Edit` chain on a brand-new file would reject the second Edit.
  The matching test at `fileReadCache.test.ts:216-228` covers it.
- **Auto-memory fix:** PR #3717 had auto-memory reads skip the cache
  entirely; this PR keeps them skipping the `file_unchanged` fast-path
  (so the freshness `<system-reminder>` always reaches the model) but
  re-enables `record` so a follow-up Edit on `AGENTS.md` doesn't
  spuriously fail. This is a real follow-on bug, good catch.
- **Compaction false-positive:** PR description acknowledges that
  Read→Edit chains spanning context compaction will eat one extra
  Read per compaction. That's a sensible cost.
- **Nit (duplication):** `requirePriorRead` exists separately on
  `EditToolInvocation` and `WriteFileToolInvocation`. PR description
  defers extraction until a third caller emerges. Reasonable, but
  consider at least sharing the error-message string constants
  (`EDIT_REQUIRES_PRIOR_READ` message text) in a single module so they
  don't drift.
- **Nit (test seed helper):** `seedPriorRead` helper (lines 100-117
  of `edit.test.ts`) is excellent and dramatically reduces churn —
  consider exporting it from a shared test util so the WriteFile
  tests can use the same shape rather than duplicating it.
- **`Config.fileReadCacheDisabled` escape hatch:** correctly threaded
  through (`getFileReadCacheDisabled` mock at `edit.test.ts:84`).
  Good for incident response.

## Rationale

Real correctness improvement, opt-out flag for incident response,
covers the new-file create-then-edit edge case in `recordWrite`,
auto-memory regression test included. Two stylistic nits, neither
blocking.
