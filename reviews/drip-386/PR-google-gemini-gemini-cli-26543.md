# google-gemini/gemini-cli#26543 — docs: update README preview image

- **Head SHA**: `e65185f5934b86164b201fee79d22425649f06fe`
- **Stats**: +1 / -1, 2 files

## Summary

Two-line change: replaces the README preview screenshot binary at `docs/assets/gemini-screenshot.png` with a fresher capture, and rewrites the alt-text from "Gemini CLI Screenshot" to "Gemini CLI terminal preview." The image swap is a binary diff so the actual visual change is unverifiable from the patch alone, but the alt-text rewrite is a small accessibility win — the new text describes what's *in* the image (a terminal preview) rather than just what kind of asset it is.

## Specific citations

- `README.md:9`: alt-text rewrite. `![Gemini CLI Screenshot](/docs/assets/gemini-screenshot.png)` → `![Gemini CLI terminal preview](/docs/assets/gemini-screenshot.png)`. The path is unchanged. The new text is six characters longer and slightly more descriptive — "terminal preview" tells a screen-reader user *what kind of view* the image conveys, where "Screenshot" only repeated the file-extension fact. Title-case → sentence-case is also more in line with most modern README alt-text conventions.
- `docs/assets/gemini-screenshot.png` (binary): file is replaced (`Binary files a/... and b/... differ`). Without `git lfs` or an attached preview I can't see the new image, so this review is taking on faith that (a) the image still depicts the CLI in a way that matches the alt-text "terminal preview," (b) the new image's filesize is reasonable for a README preview (PNG previews shipped to GitHub.com README rendering should generally stay under ~500KB to keep clones snappy), and (c) the image doesn't contain any incidental sensitive content (a typed prompt with internal info, a visible API key in a `--debug` line, an open editor showing a private path). Reviewers with the diff checked out should verify all three before stamping.
- No other files changed; the repository's i18n READMEs (per the project's prior pattern in `README.zh.md` etc., though I didn't enumerate them here) presumably either share this same English asset or have their own — not in scope for this PR.

## Verdict

**merge-after-nits**

## Rationale

Trivial cosmetic change with a small accessibility-text improvement; nothing here is risky. The single nit is process, not content: the PR description should attach a thumbnail or before/after preview of the binary image so future archaeology (`git log -p docs/assets/gemini-screenshot.png` shows nothing useful for binaries) doesn't require checking out the SHA. A maintainer doing a quick visual sanity-check should also confirm: (a) `du -k docs/assets/gemini-screenshot.png` is in a sane range — anything north of 1MB is excessive for a README preview and could be stripped with `oxipng -o4` or `pngquant`; (b) the image doesn't include any text overlays referencing internal product names, dev-machine paths, or test API keys; (c) the alt-text "terminal preview" matches the *content* of the actual capture (i.e., it really is a terminal session view, not a marketing splash). If all three pass on local checkout, ship; the diff itself is unobjectionable.
