# sst/opencode#25010 — feat(ui): Add support for RTL languages in rendered chat content and text box

- PR: https://github.com/sst/opencode/pull/25010
- HEAD: `45d1d71`
- Author: GiladKingsley
- Files changed: 4 (+26 / -9)

## Summary

Adds RTL (Arabic / Hebrew / Persian) layout support across the prompt input,
markdown render, and user-message display by sprinkling `dir="auto"` on the
right elements and migrating `padding-left` / `margin-left` /
`border-left` / `text-align: left` to logical-property equivalents
(`padding-inline-start`, `margin-inline-start`, `border-inline-start`,
`text-align: start`). Code blocks are explicitly clamped to LTR so program
text doesn't get visually mirrored.

## Cited hunks

- `packages/app/src/components/prompt-input.tsx:1347` and `:1370` —
  `dir="auto"` on the `contenteditable` editor div and the placeholder
  overlay div, so per-message language detection drives caret + glyph
  direction.
- `packages/ui/src/components/markdown.css:10` — top-level `.markdown` gets
  `text-align: start` (was implicit `left`), so paragraphs flip naturally
  inside an `dir="rtl"` ancestor.
- `packages/ui/src/components/markdown.css:64-65, 75, 100, 105, 109-111` —
  `padding-left`/`margin-left` → `padding-inline-start`/
  `margin-inline-start` for `ul`, `ol`, nested `li > ol`, blockquote.
  `border-left` → `border-inline-start` on blockquote.
- `packages/ui/src/components/markdown.css:208-209` — code-block container
  gets explicit `direction: ltr` and `text-align: left`. Correct: code is
  semantically LTR even in an RTL document; mirroring `console.log()` would
  be actively wrong.
- `packages/ui/src/components/markdown.css:246` — table-cell `text-align`
  switched `left` → `start`.
- `packages/ui/src/components/markdown.tsx:176-181, 354` — new
  `markBlockDirection` runs `dir="auto"` on each block-level child of the
  rendered markdown root, so a paragraph that mixes English and Hebrew
  gets per-block direction detection. Outer `<div data-component="markdown">`
  also gets `dir="auto"`.
- `packages/ui/src/components/message-part.tsx:1103` — user-message text
  gets `dir="auto"` so a typed Arabic prompt renders right-aligned without
  affecting English prompts.

## Risks

- `markBlockDirection` runs once per `decorate` call against the live DOM.
  If the markdown stream re-renders incrementally (token-by-token) without
  re-running `decorate`, late-arriving text may not get `dir="auto"`. Worth
  confirming the streamer triggers a `decorate` pass on every chunk, not
  just at the end.
- `direction: ltr` on code blocks is correct, but inline code (`<code>`
  not wrapped in `<pre>`) inside RTL paragraphs isn't covered here —
  `printf("hello")` mid-paragraph could still flip. A follow-up `code:not(pre code) { unicode-bidi: plaintext }` would close that gap.
- `dir="auto"` uses the first strong-directional character of the element's
  text. For a user message that starts with a code-fence ` ``` ` (no
  strong-direction chars) the heuristic falls through to LTR which is
  fine, but mixed-direction prompts that lead with punctuation can still
  surprise users — that's an upstream `dir="auto"` quirk, not this PR.
- Changing CSS to logical properties changes computed-style values. If any
  test snapshot or visual-regression suite asserts on `padding-left:32px`
  it'll break; not a correctness issue but a heads-up.

## Verdict

**merge-after-nits**

## Recommendation

Add the inline-code carve-out (`unicode-bidi: plaintext` on `:not(pre) > code`)
and confirm the streaming render path reapplies `dir="auto"` on chunks; the
core RTL plumbing is sound and the LTR clamp on code blocks is exactly right.
