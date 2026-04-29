# sst/opencode#24973 — fix(ripgrep): handle broken symlinks gracefully in files() stream

- **Repo:** sst/opencode
- **PR:** #24973
- **Head SHA:** d23cc60bab86b044da335e5492b1ad65ec8af28d
- **Base:** dev
- **Author:** community

## Summary

Two-part fix to keep `files()` streaming usable in trees that contain broken symlinks. (1) Adds `--no-messages` to the rg argv at `packages/opencode/src/file/ripgrep.ts:198` so soft I/O errors (broken symlinks, EACCES) are not written to stderr. (2) Accepts ripgrep exit code `2` alongside `0`/`1` at `packages/opencode/src/file/ripgrep.ts:363` since rg returns 2 for "found valid results plus soft errors" and the previous gate treated that as fatal, which truncated the stream and surfaced as `PlatformError` upstream. Adds a regression test at `packages/opencode/test/file/ripgrep.test.ts:172-188` that creates a real file plus a `fs.symlink(nonexistent, broken-link)` and asserts `real.txt` still appears in results.

## File:line references

- `packages/opencode/src/file/ripgrep.ts:198` — `--no-messages` added to argv
- `packages/opencode/src/file/ripgrep.ts:363` — `code === 0 || code === 1 || code === 2` gate, with comment explaining parity with `searchArgs`
- `packages/opencode/test/file/ripgrep.test.ts:172-188` — new test "files returns valid results despite broken symlinks"

## Verdict: **merge-after-nits**

## Rationale

The change is small (3 lines of behavior + 19 lines of test), surgical, and matches behavior already present in the search path (per the inline comment). The semantics align with rg's documented exit-code contract. Two nits before merge:

1. The test only asserts `expect(files).toContain("real.txt")` — it doesn't assert `broken-link` is *absent* from the results, so a future regression where rg's `--follow` started yielding the broken link as a zero-byte entry would silently pass. Add `expect(files).not.toContain("broken-link")`.
2. Consider asserting the full set is exactly `["real.txt"]` (with `--no-messages` to prove the symlink doesn't escape as a result), since `toContain` will keep passing if a future change adds extra phantom entries.
3. Optional: add a second test variant where `follow: false` to lock in that broken symlinks are simply skipped (not followed) and exit code remains `0`, distinct from the `follow: true` path that exercises the new code-2 branch.
