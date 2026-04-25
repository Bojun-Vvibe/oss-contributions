# cline/cline PR #10243 — fix: DiffViewProvider — await scroll, accurate line count, BOM preservation

@26098896 · base `main` · +42/-19 · author `bob10042`

## Summary
Three small but real correctness fixes in the diff/edit pipeline: await an async scroll, replace a regex-based line counter, and round-trip a UTF-8 BOM through save so BOM files don't appear as "user edited" diffs.

## What changed
- `src/integrations/editor/DiffViewProvider.ts:25` — adds `originalHadBom: boolean` field; set in `open()` at line 49 (`this.originalContent.startsWith("\ufeff")`); reset in `reset()` at line 515.
- `src/integrations/editor/DiffViewProvider.ts:370-377` — new `stripBom()` helper applied to `preSaveContent`, `postSaveContent`, and `this.newContent` before EOL normalization, so the post-save user-edits diff isn't tripped by a re-added BOM.
- `src/integrations/editor/DiffViewProvider.ts:451` — `revertChanges()` now uses `await this.getDocumentLineCount()` instead of `(contents.match(/\n/g) || []).length + 1`, eliminating an off-by-one with mixed CRLF/LF.
- `src/integrations/editor/DiffViewProvider.ts:476` — adds the missing `await` to `this.scrollEditorToLine(lineCount)` in `scrollToFirstDiff()`.
- `src/integrations/editor/FileEditProvider.ts:113-119` — re-prepends `\ufeff` on save when `originalHadBom && !this.documentContent.startsWith("\ufeff")`.
- Drive-by: `Promise<Boolean>` → `Promise<boolean>` across `ACPDiffViewProvider.ts:214`, `ExternalDiffviewProvider.ts:58`, `VscodeDiffViewProvider.ts:193`, `DiffViewProvider.ts:164`, `FileEditProvider.ts:107`, and the two test classes.

## Key observations
- The three fixes are tightly scoped and individually correct. The `Boolean`→`boolean` cleanup is a separate concern — would be cleaner as its own commit but the diff is mechanical and harmless.
- `stripBom` is defined inline at `DiffViewProvider.ts:370` as a one-liner closure; fine, but could live alongside `originalHadBom` as a static helper for reuse with the FileEditProvider re-add path (currently duplicated logic).
- `originalHadBom` is checked in two unrelated providers (`DiffViewProvider` for diff comparison, `FileEditProvider` for write); the protected-field plumbing is clean.
- Test coverage update in `__tests__/DiffViewProvider.test.ts` only changes type annotations — no new test for the BOM round-trip or the line-count fix. A small fixture file with BOM + CRLF would lock these in.
- `revertChanges()` change drops the local `contents` read entirely — make sure no other side effect was relying on it (looks safe in this diff window).

## Risks/nits
- BOM handling is invisible by default in editors; without a regression test these will silently drift again. Strongly request adding one.

**Verdict: merge-after-nits**
