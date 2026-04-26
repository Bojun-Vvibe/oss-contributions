# anomalyco/opencode #24364 — fix(provider): reject unsupported image mime types

- **Author:** pascalandr
- **Head SHA:** `3e7cfe29b2cda5b82645f67da429388305348794`
- **Base:** main
- **Size:** +52 / -0 (2 files)
- **URL:** https://github.com/anomalyco/opencode/pull/24364

## Summary

The provider transform layer previously forwarded any `image/*` payload
to a model that declared image input support, even MIME types most
providers reject (`image/avif`, custom spec formats). Result: opaque
provider 400s like `Input should be 'image/jpeg', 'image/png',
'image/gif' or 'image/webp'`. This PR pre-filters those payloads in
`packages/opencode/src/provider/transform.ts` and substitutes a
deterministic text error part the model can act on.

## Specific findings

- `packages/opencode/src/provider/transform.ts:12` — defines
  `SUPPORTED_IMAGE_MIME_TYPES = new Set(["image/jpeg", "image/png",
  "image/gif", "image/webp"])`. This is the OpenAI/Anthropic intersection
  of universally accepted formats; correct floor.
- `transform.ts:22-24` — new `unsupportedImageMime(mime)` helper:
  `mime.startsWith("image/") && !SUPPORTED_IMAGE_MIME_TYPES.has(mime.toLowerCase())`.
  Right shape — only fires inside the image namespace, so non-image
  attachments (`audio/*`, `video/*`, `application/pdf`) flow unchanged
  through the existing `mimeToModality` path on line 14.
- `transform.ts:309-316` — the gate is placed *before* the
  `mimeToModality(mime)` call on line 320, so an unsupported image MIME
  is replaced with a text part rather than being silently downgraded
  to `undefined` modality and dropped. Order matters here; the placement
  is correct.
- `transform.ts:310-315` — the substituted text part:
  `"ERROR: Cannot read ${name} (unsupported image format ${mime}).
  Supported image formats are JPEG, PNG, GIF, and WebP. Inform the user."`
  Two virtues: (a) the model-readable instruction "Inform the user"
  triggers the assistant to surface the failure rather than silently
  hallucinating around the missing image; (b) including both filename
  (when present) and the offending MIME makes the cause grep-able in
  rollouts. Minor: the prefix `ERROR:` is bare ASCII; many other tool
  errors in this codebase use a slightly different convention — worth a
  glance to make sure downstream parsers (e.g. log scrapers, eval
  harnesses) aren't keying on a different string.
- `transform.ts:312` — `const name = filename ? \`"${filename}"\` :
  mime;` — the quoting choice produces nicely readable messages
  (`Cannot read "spec.avif"` vs `Cannot read image/avif`). The fallback
  to MIME-only when filename is absent is the right minimum.
- `packages/opencode/test/provider/transform.test.ts:1058-1075` —
  `should replace unsupported image mime types with error text` test
  pins the exact substituted text for a `data:image/avif;base64,Zm9v`
  inline image. Test is precise — it asserts on the full text body, not
  just the `type === "text"` shape, so any future drift in the message
  is caught.
- `transform.test.ts:1077-1095` — second test pins the file-shaped
  variant with filename `spec.avif`, including the quoted-filename form.
  Both branches of the `name` ternary are covered.
- **Gap:** no test asserts the *negative* case — that a supported
  format (e.g. `image/png`) still flows through unchanged after this
  patch. The closest existing test is `should handle mixed valid and
  empty images` further down, which doesn't exercise the new gate
  explicitly. Worth a one-liner regression test to lock in that the gate
  doesn't false-positive.
- **Gap:** uppercase MIME (`IMAGE/PNG`) is normalized via
  `.toLowerCase()` inside the gate, but `mimeToModality` on line 14
  uses raw `mime.startsWith("image/")` without lowercasing. So a
  hypothetical `IMAGE/AVIF` MIME would hit the new gate (it's
  lowercased before `Set.has`) — but `unsupportedImageMime` itself
  starts with `mime.startsWith("image/")` (lowercase only). So
  `IMAGE/AVIF` would *not* be caught by the gate and would fall through
  to the original "drop" path. Tighten by calling `.toLowerCase()` on
  the `startsWith` check too, or explicitly say MIMEs are assumed
  pre-normalized at this layer (and add a comment).

## Verdict

`merge-after-nits`

## Reasoning

The bug being fixed is real and high-value (silent provider 400s on
attachments are a known support-ticket source per the linked #17772
and #15264). The implementation is small, well-placed, and the
substituted error message has the right model-actionable shape. Tests
cover both branches of the new code path.

The two gaps — no positive-case regression test for supported formats,
and inconsistent MIME-case handling between `unsupportedImageMime` and
`mimeToModality` — are nits worth addressing before merge but neither
blocks the fix from being correct on the hot path.
