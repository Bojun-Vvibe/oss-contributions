# charmbracelet/crush PR #2785 — fix: limit view size checks to returned content

- URL: https://github.com/charmbracelet/crush/pull/2785
- Head SHA: `fa1acff88d05871ee16240322f5d818acf08c0ef`
- Author: taoeffect (Greg Slepak)

## Summary

Reworks the file-size check in `internal/agent/tools/view.go` so the
`MaxViewSize` limit applies to the **returned** content section
(post offset/limit) rather than the raw file size on disk. Removes the
early `fileInfo.Size() > MaxViewSize` rejection (line ~163 of the prior
code), keeps the size guard for image files, and threads a new
`maxContentSize` parameter into `readTextFile()` that enforces the cap
line-by-line. A new `contentTooLargeError{Size, Max}` type carries the
breach upward and is unwrapped via `errors.As`.

## Review

This is a real usability fix. Today, any 250 KB log file is unreadable
even with `offset=10000, limit=200`, which defeats the whole purpose of
having pagination. The new shape — "raw size unbounded, but the slice you
get back is capped" — matches what every other agent file-view tool does.

Implementation looks clean:

- `internal/agent/tools/view.go:171` — drops the early `fileInfo.Size()`
  guard for non-image files, keeps it for images (with a clearer error
  message). Good split.
- `view.go:200` — `maxContentSize := MaxViewSize; if isSkillFile {
  maxContentSize = 0 }` correctly preserves the existing "skills are
  unlimited" exception by treating `0` as "no cap".
- `view.go:~293` — the projected-size check inside the scanner loop adds
  `+1` when there's already a line (for the `\n` join). Make sure the
  final assembly path also joins with `\n` and not `\n\n`, otherwise the
  cap is off by `len(lines)-1` bytes.
- `lines := make([]string, 0, min(limit, DefaultReadLimit))` is a
  worthwhile bounded-prealloc fix (was `make([]string, 0, limit)` which
  would happily try to allocate a slice header for `limit=2_000_000`).
- `view.md` doc tightens the "Max returned file content section: 200KB
  after offset/limit are applied" wording — good. Drops `BMP` and `SVG`
  from the supported-image list; that's a behavior narrowing — confirm
  it matches `getImageMimeType` which the diff doesn't show.

Nit: `contentTooLargeError.Size` reports the *projected* size at the
moment the cap was breached, which is fine for the error message. But
the user-facing string says "Content section is too large (%d bytes)"
which suggests an actual measured size. Consider phrasing it as
"would exceed maximum" so it's not surprising when the number is just
shy of the next line break.

## Verdict

`merge-after-nits` — fix is well-targeted; double-check the BMP/SVG
removal in `view.md` matches `getImageMimeType` and tighten the error
phrasing.
