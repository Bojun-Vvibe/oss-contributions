# charmbracelet/crush #2785 — fix: limit view size checks to returned content

- **PR:** https://github.com/charmbracelet/crush/pull/2785
- **Head SHA:** `fa1acff88d05871ee16240322f5d818acf08c0ef`
- **Author:** taoeffect
- **Size:** +208 / -15 across 3 files
- **Closes:** #2784, #2756
- **AI disclosure:** fix applied by GPT-5.5

## Summary

The View tool's size guard was rejecting large files even when the caller asked for a small offset/limit window. After this PR the guard runs on the *returned* slice, so a 50 MB log can still be viewed in 200-line chunks. Image files keep the existing whole-file size cap. Skill files are exempted from the content cap (they need to be loaded whole).

## Specific references

- `internal/agent/tools/view.go:60-70` — new typed error:
  ```go
  type contentTooLargeError struct { Size int; Max int }
  ```
  Lets the call site distinguish "too big" from generic IO errors via `errors.As`. Clean.
- `view.go:182-186` — image-file branch keeps the *file size* check (`fileInfo.Size() > MaxViewSize`), correct because images are read whole regardless of offset/limit.
- `view.go:200-216` — text-file path computes `maxContentSize := MaxViewSize`, but sets `maxContentSize = 0` for `isSkillFile`, then passes it into the new `readTextFile(filePath, params.Offset, params.Limit, maxContentSize)` signature. The `0` sentinel for skill files is worth a code comment — currently a reader has to trace into `readTextFile` to see "0 means unlimited."
- `view.go:275-300` — `readTextFile` now tracks `contentSize` per appended line (`projectedSize := contentSize + len(lineText); if len(lines) > 0 { projectedSize++ }` to account for joining `\n`). Returns the typed `contentTooLargeError` when projected size exceeds `maxContentSize`. Off-by-one for the joining newline is handled correctly.
- `view_test.go` — regression tests exercising offset/limit windows on large files plus the skill-file exemption.

## Concerns

1. **`maxContentSize == 0` as "unlimited"** is a magic number. Either a named constant (`unlimitedViewSize = 0`) or a `*int` would self-document better. Worth a 1-line comment at minimum.
2. **Error message duplication** — the inline string at `view.go:67` ("content section is too large (%d bytes). Maximum size is %d bytes") and the call-site fallback at `view.go:213` are nearly identical. Could be sourced from `tooLarge.Error()` directly to keep them in sync.
3. **Skill files exempt from the cap** is a reasonable policy choice but should be documented in `view.md` so users understand why a 100 MB skill file would load while a 100 MB regular file with no offset would not.

## Verdict

**merge-after-nits** — the underlying fix is correct and well-tested. Nits are docstring/readability only.
