---
pr: https://github.com/charmbracelet/crush/pull/2757
head_sha: 3a3b1a82ae74e2f76ada69d91d68bb95cf80587b
author: meowgorithm
additions: 61
deletions: 0
files_touched: 2
---

# Scope

Fixes the image-handling path in the `view` tool so it determines an
image's MIME type by content-sniffing the file's magic bytes, rather
than trusting the file extension. Motivation: providers like Anthropic
strictly validate the declared media type against the base64 magic
bytes and 400 on mismatch; a `.png` that actually contains JPEG bytes
currently breaks the request. Adds a focused unit test table.

# Specific findings

- `internal/agent/tools/view.go:192-198` (head `3a3b1a8`) — the call
  site now does `mimeType = sniffImageMimeType(imageData, mimeType)`
  *after* the existing extension-derived `mimeType` is computed, so the
  sniffed value only wins when it identifies a supported format. Good:
  the fallback preserves the prior behavior for sniff-resistant inputs.
- `internal/agent/tools/view.go:325-342` — `sniffImageMimeType` uses
  `http.DetectContentType` and accepts only `image/jpeg`, `image/png`,
  `image/gif`, `image/webp`. Two notes:
  - `http.DetectContentType` *does* recognize `image/bmp` and
    `image/svg+xml` (the latter explicitly handled by the `;` strip
    comment). The current allowlist excludes both. If the rest of the
    `view` tool already forbids those formats upstream, fine; otherwise
    a `.bmp` file would be sent with the extension-derived type and
    likely still fail at the provider. Worth either expanding the
    allowlist or adding a comment referencing where unsupported
    formats are filtered.
  - The `;` strip is described as defensive — accurate for current
    Go stdlib behavior, but a brief reference to net/http's docs
    (which explicitly say image sniffers return bare types) in the
    comment would help future readers know whether the strip is dead
    code or future-proofing.
- `internal/agent/tools/view_test.go:140-171` — the table covers
  jpeg-in-png, png-in-jpg, gif, webp, matching, unsniffable, and empty.
  Solid. Suggest adding a negative case where the sniffed type is e.g.
  `text/plain; charset=utf-8` (HTML/text masquerading as an image
  extension) to lock the fallback behavior — currently
  `"unsniffable content falls back"` (random text) covers this in
  spirit but the assertion is on the fallback value, not on what
  `http.DetectContentType` returned, so the test would still pass if
  the function silently propagated a non-image MIME.
- The change is internal-only, no public API surface changes.

# Suggested changes

1. Decide whether `image/bmp` (and possibly `image/svg+xml`, where the
   provider supports it) should be in the allowlist; add a comment
   pointing to where unsupported formats are otherwise rejected.
2. Add an explicit unit test asserting that a non-image `DetectContentType`
   result (e.g. `text/plain`) is *not* propagated and the fallback is
   returned unchanged.
3. Microscopic: the comment at line 326 references a tool name
   ("pinchtab") that downstream readers won't recognize — consider a
   more generic phrasing like "some tools save files with a mismatched
   extension".

# Verdict

`merge-as-is`

# Confidence

High — small, well-scoped fix with good test coverage; suggestions are
all polish.

# Time spent

~7 minutes.
