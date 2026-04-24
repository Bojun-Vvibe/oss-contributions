# PR #19292 — reject unsupported `js_repl` image MIME types early

**Repo:** openai/codex • **Author:** etraut-openai • **Status:** merged
• **Net:** +118 / −5

## Change

Add an allowlist (`image/png`, `image/jpeg`, `image/webp`,
`image/gif`) at two boundaries:

1. JS kernel `codex.emitImage(...)` — both the `{bytes, mimeType}`
   path (`encodeByteImage`) and the `data:` URL path
   (`normalizeEmitImageUrl`) now call a shared `assertEmitImageMimeType`.
2. Rust host `validate_emitted_image_url` parses the `data:` URL and
   matches against the same allowlist (case-insensitive).

The user-facing instruction string in `agents_md.rs` is updated to
spell out the supported encodings. Tests cover (a) byte-path with
`image/rgba` is rejected, (b) URL-path bad MIME returns the typed
error, (c) case-insensitive `DATA:image/PNG` is accepted.

## Why this is load-bearing

The PR description nails the failure mode: an unsupported MIME like
`image/rgba` would propagate to the model-input path and trip the
output sanitizer's `debug_assert!` that's meant as a final safety
net. In debug builds that's a panic; in release builds the same
event becomes a silent corruption (image_url field set to
`data:image/rgba;base64,...` which the upstream Responses API
rejects 400, killing the turn). A wide-deployed agent calling
`page.screenshot({ type: 'jpeg' })` and accidentally passing the
raw RGBA buffer would lose every single tool turn.

## Defense-in-depth audit

The two-layer check (JS kernel + Rust host) is correct here, not
redundant:

- JS kernel rejection produces a clean `Error` the model sees and
  can self-correct from in the next turn.
- Rust host rejection catches the case where someone bypasses the
  kernel — direct freeform `js_repl` input that constructs a tool
  output object containing an unsupported MIME, or future tool
  surfaces that route through the same `validate_emitted_image_url`
  path.

The shared allowlist constant should be promoted to one canonical
location, though. Right now it's duplicated:

- `kernel.js`: `SUPPORTED_EMIT_IMAGE_MIME_TYPES` array
- `mod.rs`: a `matches!` arm with the same four strings inline

If a fifth format (AVIF, BMP) is ever added, both must change in
lockstep. Worth either generating the JS list from a shared `.json`
fixture at build time, or adding a one-line cross-reference comment.

## Sharp edge: the case-insensitive parse

`parseDataUrlMimeType` returns the media type **as written** and
`assertEmitImageMimeType` lowercases it before comparison. The Rust
side calls `to_ascii_lowercase()` and matches directly. Both correct.
But neither side handles `data:image/png; charset=utf-8;base64,...`
— the `.split(';').next()` call grabs the type before any parameter,
which is what RFC 2397 expects. Good.

What's *not* handled: the bare `data:base64,...` (no MIME — defaults
to `text/plain` per RFC 2397). The kernel-side `parseDataUrlMimeType`
treats `""` as falsy and throws "expected image data URL to include
a MIME type" — correct behavior. Rust side also requires
`mediaType` to be non-empty — also correct. The error messages are
slightly inconsistent: JS says "expected image data URL to include
a MIME type", Rust says "expected a valid image data URL". Worth
unifying for the user-facing experience.

## Concrete next step

1. Promote `SUPPORTED_EMIT_IMAGE_MIME_TYPES` to a single source of
   truth (build-time generated, or at minimum a comment linking the
   two definitions).
2. Unify the JS / Rust error messages so model self-correction sees
   the same string regardless of which boundary caught the issue.
3. Add a release note: any in-the-wild agent that emitted
   `image/svg+xml` or `image/bmp` (the two most plausible mistakes)
   will now hard-error instead of silently 400ing — that's a
   net-positive change, but observable.

## Verdict

Correct, well-tested, and closes a debug-panic / release-silent-fail
seam. The duplication of the allowlist is a minor maintenance
concern; the unification of error messages is a small UX polish.

## What I learned

When a sanitizer is documented as "final safety net," any path that
relies on it for primary validation is a latent panic in debug
builds. The right fix is to move validation to the entry point and
let the sanitizer remain a `debug_assert!` for invariants, not user
input. This PR is exactly that pattern.
