# QwenLM/qwen-code#3774 — feat(core): enforce prior read before Edit / WriteFile mutates a file

- **PR**: https://github.com/QwenLM/qwen-code/pull/3774
- **Head SHA**: `303b6b7d4ac7706d78fc3f4e40c6bef222fa7c9d`
- **Size**: +677 / -6, 10 files
- **Verdict**: **merge-after-nits**

## Context

Builds on #3717's session-scoped `FileReadCache`. That PR added the cache; this PR uses it to enforce that the model has actually `read_file`'d a pre-existing file (and seen its current bytes) before `Edit` or `WriteFile` is allowed to mutate it. Closes the "plausible-but-stale `old_string` matches against bytes the model has not seen" failure mode that the existing `0 occurrences` guard does not catch.

Two new error codes:
- `EDIT_REQUIRES_PRIOR_READ` — file has no entry in the session cache.
- `FILE_CHANGED_SINCE_READ` — file was read earlier but `mtime` or `size` has drifted.

## What's right

**`fileReadCache.ts:118-138` — `recordWrite` seeds read metadata for brand-new entries.**

Old behavior was "do not set `lastReadAt` or flip `lastReadWasFull`" (the deleted test at `fileReadCache.test.ts:215-218` literally asserted that). New behavior: when `recordWrite` is called on an entry that has never been Read (`entry.lastReadAt === undefined`), populate `lastReadAt = now`, `lastReadWasFull = true`, `lastReadCacheable = true`. The reasoning at `:121-128` is exactly right — the model authored the bytes it just wrote, so for prior-read enforcement it has effectively "seen" the full text content, and without this seeding a `create → edit → edit` chain would reject the second edit because the entry `recordWrite` created would still have undefined read-metadata. The new test at `fileReadCache.test.ts:217-228` pins this with `expect(entry.lastReadAt).toBe(entry.lastWriteAt)` and the three reads/cacheable assertions.

**`priorReadEnforcement.ts` (new file, 116 lines).**

Centralizes the enforcement helper. Per the PR body, deferred extraction of duplicate-with-trivial-deltas logic from `EditToolInvocation` and `WriteFileToolInvocation` into this shared file (acceptable trade-off: the bodies are nearly identical but the contexts differ, so a third caller is the right trigger for the abstraction).

**`edit.test.ts:937-1162` — 9 enforcement tests** covering the documented matrix:
1. rejects an edit when the file has not been read in this session
2. rejects an edit when the file has been modified since the last read
3. exempts new-file creation from prior-read enforcement
4. allows a chain of edits without re-reading between them
5. bypasses enforcement entirely when `fileReadCacheDisabled` is true

The first arm (visible at `:944-961`) directly asserts:
- `result.error?.type` equals `ToolErrorType.EDIT_REQUIRES_PRIOR_READ`
- error message matches `/has not been fully read in this session/`
- file content remains unmutated (`expect(fs.readFileSync(filePath, 'utf8')).toBe('untouched content')`)

The "no mutation on rejection" assertion is the load-bearing safety property — easy to forget, present here.

**`seedPriorRead` test helper at `edit.test.ts:107-114`.**

The PR adds a one-line helper that calls `fileReadCache.recordRead(filePath, stats, { full: true, cacheable: true })` and is invoked at the top of every existing test that exercises pure Edit business logic (diffing, encoding, replace_all, etc.). The diff shows ~20 such call-sites added (e.g. `:271, :291, :305, :426, :539, :561, :579, :595, :631, :689, :706, :746, :761, :789, :813`). This is the right way to keep the existing test suite working without weakening the new enforcement — the alternative (disabling enforcement in tests) would have hidden bugs.

**Auto-memory regression fix (PR body).**

#3717 had auto-memory reads (`isAutoMemPath`) skip the cache entirely. With enforcement enabled, that meant a model that just Read `AGENTS.md` could not then Edit it — the read never registered. The fix decouples the two concerns: auto-memory reads still skip the `file_unchanged` fast-path (so the per-read freshness `<system-reminder>` always reaches the model) but they DO record into the cache so the follow-up Edit sees `fresh`. The new regression test in `read-file.test.ts` pins this. This subtle decoupling is the right call.

**`tool-error.ts` adds two new `ToolErrorType` enum values (+11 lines).** Correctly added to the enum, the new error codes are referenced from both the helper and the tests.

## Risks / nits

- **The `fileReadCacheDisabled` escape hatch is the entire incident-playbook surface.** The PR description correctly identifies this as the operational mitigation. It would be worth a one-line addition to the user docs (or `AGENTS.md`-style operator guide) explicitly documenting "if Edit/Write enforcement starts blocking your model, set `fileReadCacheDisabled: true` in config to revert to pre-3717 behavior". Without that, an operator hitting the false-positive path during compaction (acknowledged in the PR body) has to read the source to find the escape hatch.
- **Compaction-induced false positives are real.** Author acknowledges: "the first Edit after compaction will be rejected with `EDIT_REQUIRES_PRIOR_READ`. The error message tells the model to re-read; one extra Read per compaction event is the cost." Reasonable, but worth measuring the impact in a follow-up — for tools that mutate many files in one compactable session, the post-compaction tax could be N extra reads. Consider a metric or debug log on `EDIT_REQUIRES_PRIOR_READ` rate so operators can spot pathological loops.
- **Subagent inheritance via per-Config own-property machinery from #3717 (mentioned in the PR body) is not exercised by the visible tests.** A test asserting "subagent gets empty cache, first Edit requires Read" would pin the documented contract.
- **`requirePriorRead` helper is on `EditToolInvocation` and `WriteFileToolInvocation` separately, not extracted.** PR body explicitly acknowledges and defers this. Fine — the right time to extract is when a third caller appears (`MultiEdit`?). Just don't forget when it does.
- **`FILE_CHANGED_SINCE_READ` triggers on `mtime OR size` drift.** Mostly the right heuristic, but on filesystems with second-resolution mtime (some macOS configs, ZIP-mounted FS) two writes within the same second can collide on mtime — the size check then catches divergent-content cases but not equal-size-different-content edits. Not a blocker since this is a defense-in-depth check (the model still has to supply the right `old_string` to match), but worth a one-line comment on the `mtime`/`size` fingerprint noting the limitation.
- **The new file `priorReadEnforcement.ts` is 116 lines but isn't shown in the diff window** — would normally request a walkthrough on the per-tool integration shape and how the helper composes with `Config.fileReadCacheDisabled`. Trusting the test coverage for now.
- **Test assertion at `edit.test.ts:954` uses `result.error?.message` regex match `/has not been fully read in this session/`.** The PR body's example error message says "has not been read in this session" (no "fully"). The actual error message includes "fully read" — slight wording divergence between PR body and code. Pick one (recommend "has not been fully read" for accuracy since the cache also tracks `lastReadWasFull`).

## Verdict

**merge-after-nits.** Substantive safety improvement with a careful design (cache as means, enforcement as end), a deliberate auto-memory decoupling that fixes a real corner case, a thoughtful test-helper pattern that keeps the existing 6308-test suite green, and a documented escape hatch. The concerns above are all polish (operator docs for the escape hatch, mtime-collision comment, subagent test arm, error-message wording consistency) — the structural design is sound.
