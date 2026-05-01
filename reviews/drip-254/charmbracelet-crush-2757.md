# charmbracelet/crush #2757 — fix(tools/view): detect image mime type; don't rely on extension

- **Repo:** charmbracelet/crush
- **PR:** https://github.com/charmbracelet/crush/pull/2757
- **HEAD SHA:** `3a3b1a82ae74e2f76ada69d91d68bb95cf80587b`
- **Author:** meowgorithm
- **Verdict:** `merge-as-is`

## What the diff does

Closes a "tool returns 400 from Anthropic when image extension and
content disagree" bug in the View tool's image-attachment path.
Some upstream tools (the PR mentions `pinchtab`) write JPEG bytes
to a file with a `.png` extension, and the prior `getImageMimeType`
extension-only logic at `internal/agent/tools/view.go:313-322` would
report `image/png` to the model provider despite the bytes being
JPEG-magic. Anthropic strictly validates the declared media type
against base64 content sniffing and 400s on mismatch.

Fix: introduce `sniffImageMimeType(data []byte, fallback string)
string` at `view.go:328-341` that runs `http.DetectContentType(data)`,
strips any `;`-delimited parameters defensively, and returns the
sniffed type if it's one of `image/jpeg`, `image/png`, `image/gif`,
or `image/webp` — otherwise returns the extension-derived fallback.
Wired into the image-read branch at `view.go:189-198` *after* the
file is read, so the declared media type sent to the model matches
the actual bytes.

Files of note:

- `internal/agent/tools/view.go:189-198` — call site. After
  `os.ReadFile`-equivalent successfully populates `imageData`,
  the line `mimeType = sniffImageMimeType(imageData, mimeType)`
  overrides the extension-derived `mimeType` with the
  content-sniffed value. The `mimeType` going into
  `fantasy.NewImageResponse(imageData, mimeType)` is now
  always the bytes-truth, falling back to extension only when
  sniffing can't identify the bytes.
- `internal/agent/tools/view.go:328-341` — `sniffImageMimeType`
  helper:
  - `http.DetectContentType(data)` is the right Go-stdlib
    primitive — implements the WHATWG MIME-sniffing algorithm
    against the first 512 bytes, returns `application/octet-
    stream` for unknown content (which is *not* in the allowlist
    so falls back).
  - The `if i := strings.IndexByte(sniffed, ';'); i >= 0 {
    sniffed = strings.TrimSpace(sniffed[:i]) }` defensive strip
    handles the case where `DetectContentType` returns a MIME
    type with a parameter (e.g., `image/svg+xml; charset=utf-8`).
    Current image sniffers in stdlib return bare types but the
    defensive strip future-proofs against stdlib changes.
  - The `switch sniffed { case "image/jpeg", "image/png",
    "image/gif", "image/webp": return sniffed }` allowlist is
    the right shape: if the sniffer identifies a *supported*
    image format, trust it; otherwise fall back to the
    extension-derived type. This avoids both false-positive
    over-trust on weird sniffed outputs and false-negative
    under-trust on legitimate format mismatches.
- `internal/agent/tools/view_test.go:140-171` — table-driven
  test `TestSniffImageMimeType` with 7 cases:
  - jpeg-magic-bytes-in-`.png`-fallback → `image/jpeg`
    (the load-bearing case for the original bug shape)
  - png-magic-bytes-in-`.jpg`-fallback → `image/png` (the
    symmetric case)
  - gif-magic-bytes (`GIF89a`) → `image/gif`
  - minimal RIFF/WEBP header → `image/webp`
  - matching extension and content (`pngMagic` + `image/png`
    fallback) → `image/png` (no-op confirmation)
  - unsniffable random text + `image/png` fallback → `image/
    png` (fallback path)
  - empty content + `image/jpeg` fallback → `image/jpeg`
    (empty-data path)

## Why it's right

The diagnosis nails the exact failure mode: extension-based MIME
type guessing is unreliable when upstream tools write mismatched
extensions, and providers that strictly validate the declared
media type against the actual bytes (Anthropic, possibly others)
will 400 on mismatch with no way for the model or the user to know
the file was being misreported.

`http.DetectContentType` is the canonical Go-stdlib answer for
content-sniffing — it runs the WHATWG algorithm (the same one
browsers use) on the first 512 bytes and returns a MIME type. For
the four supported image formats (jpeg/png/gif/webp) this is
authoritative because each format has unambiguous magic bytes at
the start (FF D8 FF for JPEG, 89 50 4E 47 for PNG, GIF87a/GIF89a
for GIF, RIFF...WEBP for WebP).

The allowlist-on-sniffed approach is the right conservatism: if
the sniffer identifies a supported image format, override the
extension; if it identifies *anything else* (`text/plain`,
`application/octet-stream`, `image/svg+xml`, etc.), keep the
extension-derived fallback. This prevents the function from
incorrectly *down-grading* a legitimate `.png` whose 512-byte
prefix doesn't sniff cleanly to `application/octet-stream` —
fallback wins in the ambiguous case, and the prior behavior is
preserved.

The `;`-strip defensive at `:336-339` handles the SVG-with-charset
edge case (which would currently fall back via the allowlist
anyway, but the strip is correct future-proofing if stdlib starts
returning parameterized image types).

The test surface is appropriately broad and dispositive:
- The two original-bug-shape cases (jpeg-in-png-extension, png-in-
  jpg-extension) directly assert the fix.
- The four positive-format cases (jpeg/png/gif/webp) lock the
  allowlist.
- The matching-extension case (`pngMagic` + `image/png`) confirms
  the no-change-on-correct-input property.
- The two fallback cases (random text, empty data) lock the
  fall-back-to-extension semantics.

Hand-rolled magic-byte fixtures (`jpegMagic`, `pngMagic`,
`gifMagic`, `webpMagic`) are the right test inputs — they're
minimal, dependency-free, and unambiguously trigger the sniffer's
format-detection arms. The webp fixture properly includes the
required `RIFF` header + length + `WEBP` + `VP8 ` chunks plus
padding.

The comment at `:191-196` ("Some tools save files with a mismatched
extension (e.g. pinchtab writes JPEG bytes to a .png file).
Providers like Anthropic strictly validate the media type against
the base64 magic bytes and 400 on mismatch, so prefer the sniffed
type whenever it identifies a supported image format.") names the
real-world symptom *and* the architectural reason for the fix, so a
future reader investigating "why don't we just trust the extension"
gets the answer immediately.

The doc comment at `:325-331` on the helper itself is similarly
load-bearing — names the fallback semantic and the provider-strict-
validation reason explicitly.

## Nits

None worth blocking. The change is well-scoped, well-tested,
well-commented, and uses the canonical stdlib primitive. Minor
beats:

- The 4-format allowlist matches what most providers
  accept, but if `image/heic` / `image/avif` start being supported
  upstream, those would need to be added here. Out of scope for
  this PR.
- `http.DetectContentType` requires reading the first 512 bytes;
  the call site at `view.go:189` is post-`ReadFile` so the entire
  file is already in memory — no concern, but if a future change
  switches to streaming reads, sniffing needs a separate
  prefix-buffer step.
- The PR description doesn't mention the `image/heic`/`avif`
  scope decision but the test list and allowlist together make
  the scope clear. No action needed.

## Verdict rationale

Right diagnosis (extension-based MIME guessing fails when upstream
writes mismatched extensions, providers strictly validate against
bytes), right primitive (`http.DetectContentType` — canonical
stdlib WHATWG sniffer), right shape (allowlist-on-sniffed with
extension fallback for unidentified-content conservatism), right
test surface (7 cases covering the original bug shape, the
symmetric case, the four supported formats, the no-change case,
and both fallback cases), well-commented at both the call site and
the helper, well-scoped (one helper, one call-site change, one
test file). Single maintainer-authored PR with the architectural
reason for the fix named explicitly in source comments.

`merge-as-is`
