# sst/opencode #25794 — docs: add Persian (Farsi) README translation

- **Head SHA:** `d70c86e40785f2a5c25d6be963818e999922edff`
- **Base:** `dev`
- **Author:** xoxxel
- **Size:** +139 / −0 (single new file `README.fa.md`)
- **Verdict:** `merge-after-nits`

## Summary

Adds a Persian (Farsi) translation of the project README following the established
multi-language pattern. Pure docs add — one new file, no code touched, no link
table reordering on the source `README.md`.

## What's right

- Standard "language-suffix README" pattern (`README.fa.md`) consistent with the 22
  existing translations enumerated in the language nav at `README.fa.md:18-39`.
- Self-references its own entry at `README.fa.md:39` so the nav block can be copied
  verbatim into the canonical English README and other translations during the
  next link-table sync.
- Right-to-left script content, numerals, and emoji renders correctly inline; image
  references and code fences left untouched (no LTR/RTL pollution into shell
  snippets at `README.fa.md:46-58`).
- Install/agent/docs/contributing/FAQ section ordering matches the canonical
  English README so downstream readers can cross-reference 1:1.

## Concerns / nits

1. **Language-list nav is missing a separator before the new entry.** At
   `README.fa.md:38` the prior line ends `<a href="README.vi.md">Tiếng Việt</a>`
   without a trailing `|`, then line 39 adds `<a href="README.fa.md">فارسی</a>`.
   Every other entry in the same block uses ` |` between siblings — pattern
   should be preserved so the rendered nav doesn't visually merge "Tiếng Việt"
   into "فارسی". Add ` |` at the end of line 38.
2. **Typos in the Farsi prose** flagged by the existing PR template ("How did
   you verify your code works? Checked markdown formatting manually" — no
   native-speaker review claimed):
   - `README.fa.md:13` `وضعبت ساخت` should be `وضعیت ساخت` (build status —
     missing yodh).
   - `README.fa.md:67` `اراده دهنده ای` should be `ارائه‌دهنده‌ای` (provider —
     misspelling + missing ZWNJ).
   - `README.fa.md:68` `رئی` should be `روی` (on — wrong consonant).
   - `README.fa.md:73` `ه جمع ما ملحق شوید` is missing the leading `ب` →
     `به جمع ما ملحق شوید` (join our community).
   - `README.fa.md:67` `لوکال` (transliteration of "local") — prefer the native
     `محلی` for consistency with the rest of the doc which uses Persian terms.
3. **Code-block direction mark missing.** Markdown headings written in Persian
   (e.g. `### نصب` at `README.fa.md:43`) are followed by an LTR fenced code
   block; on some renderers (GitHub honors this, but mirrors / docs tooling may
   not) this can flip the surrounding heading numerals. Adding an explicit
   `<div dir="rtl">…</div>` wrapper around the prose paragraphs (not the code
   blocks) would be more portable. Not blocking — GitHub renders correctly
   today.
4. **The `[!توجه]` admonition at `README.fa.md:60`** is a translation of
   `[!NOTE]`. GitHub-flavored markdown only recognizes the literal English
   tokens `NOTE | TIP | IMPORTANT | WARNING | CAUTION` — translating the token
   itself silently degrades the admonition to a plain blockquote. Keep the
   English token (`> [!NOTE]`) and translate only the body text underneath.
   This is the only correctness issue of the four; the rest are typos.
5. **No update to `README.md` language-nav.** If the project's convention is for
   each translation PR to also append `<a href="README.fa.md">فارسی</a>` to the
   language nav of the canonical English README (and ideally of every sibling
   `README.<lang>.md`), this PR is incomplete and Persian readers will only
   reach the new file by direct URL or repo browsing. Recommend either
   appending to `README.md` in this PR, or filing a follow-up that fans the
   new entry across the existing `README.*.md` siblings.

## Verdict rationale

Translation PRs to a public OSS README should land — the alternative is no
Persian-language entry point for ~110M Persian speakers. Issue 4 (broken
admonition) is a one-line fix the author can make without re-reviewing the
whole translation; issues 1-3 are wording polish; issue 5 is a navigation-fan
gap that's typically acceptable to land separately. None of these block merge,
hence `merge-after-nits` not `request-changes`.
