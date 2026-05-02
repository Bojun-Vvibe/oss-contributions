# charmbracelet/crush PR #2757 — fix(tools/view): detect image mime type; don't rely on extension

- **PR**: https://github.com/charmbracelet/crush/pull/2757
- **Head SHA**: `3a3b1a82ae74e2f76ada69d91d68bb95cf80587b`
- **Size**: +61 / −0 across 2 files (`internal/agent/tools/view.go`, `internal/agent/tools/view_test.go`)
- **Verdict**: **merge-as-is**

## What changed

`view.go` adds a `sniffImageMimeType(data []byte, fallback string) string` helper (lines ~325–342) that calls `http.DetectContentType`, strips any `; charset=…` parameter defensively, and returns the sniffed value only if it is one of `image/jpeg`, `image/png`, `image/gif`, `image/webp`; otherwise returns the supplied fallback (the extension-derived MIME). The helper is wired into the existing image-read path at line ~192: after `os.ReadFile` of the image, `mimeType = sniffImageMimeType(imageData, mimeType)` runs before the bytes are handed to `fantasy.NewImageResponse`. The new test (`view_test.go:139`) is a table-driven `TestSniffImageMimeType` covering: JPEG bytes in a `.png` fallback → `image/jpeg`; PNG bytes in a `.jpg` fallback → `image/png`; GIF magic; minimal RIFF/WEBP header; matching content; unsniffable random bytes; and empty content (both fall back).

## Why this matters

Anthropic's vision API strictly validates the `media_type` declared in a base64 image part against the actual magic bytes and returns 400 on mismatch. Users renaming JPEGs to `.png` (common when browsers save WebP-served assets, or when tools like pinchtab — the example in the in-code comment — write JPEG bytes to a `.png` file) used to break crush's `view` tool against Anthropic with an opaque 400, even though the file was readable and the bytes were a valid image. Sniffing solves it cleanly: the on-wire `media_type` now matches the on-wire bytes, and Anthropic accepts the request. Other providers that don't validate this strictly are unaffected because the sniffed result is still a real correct MIME for the bytes. Falling back to the extension when sniffing fails (random bytes, empty data, exotic format) preserves prior behavior.

## Specific call-outs

- `sniffImageMimeType` correctly limits the sniff result to a known-supported allowlist (`image/jpeg|png|gif|webp`). This avoids surfacing weirder sniffer outputs (e.g. `image/svg+xml; charset=utf-8`, `application/octet-stream`) to providers that wouldn't accept them. Good defensive scoping.
- The `if i := strings.IndexByte(sniffed, ';'); i >= 0` parameter strip is defensive but realistic — `http.DetectContentType` does append parameters for some media types. Keeping it is the right call even if current image sniffers don't trigger it.
- The `getImageMimeType(filePath)` extension-based helper at line 313 is left untouched and now serves as the *fallback* path. The two-stage "extension first, then sniff" ordering means there's no behavior change for files where extension and content already agree, which is the overwhelming majority — zero risk for existing users.
- The test uses real magic bytes (JPEG `0xff 0xd8 0xff 0xe0 …`, PNG signature, GIF89a, RIFF/WEBP), not synthetic placeholders — so it's actually exercising the real `http.DetectContentType` codepath. Comprehensive.
- The in-source comment block at line ~189 names "pinchtab" as the motivating example. Slightly editorial but useful provenance for future readers; keeping it.
- No public API changes; helper is package-private.

## Verdict rationale

Small, focused, well-tested bug fix for a real interop problem against a major provider. The helper is defensively bounded (allowlist of known image types, strips MIME params, falls back rather than guessing), the integration point is a one-line change, and the test matrix covers all four supported types plus the fallback paths. Nothing to negotiate — ship it.
