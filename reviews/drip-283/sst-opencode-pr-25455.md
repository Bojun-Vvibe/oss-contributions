# sst/opencode PR #25455 — feat(ui): add dir="auto" for bidirectional text support

- Head SHA: `f5ca46f7a471e72309cca93830576a7592587976`
- Size: +3 / -1, 3 files

## Specific refs

- `packages/app/src/components/prompt-input.tsx:1347` — adds `dir="auto"` on the `contenteditable` prompt input. Sits next to existing `aria-multiline`/`aria-label` so the browser detects strong-character direction per element. No JS change.
- `packages/ui/src/components/markdown.tsx:340` — adds `dir="auto"` on the markdown root `<div data-component="markdown">`. Applies to assistant message bodies regardless of source language.
- `packages/ui/src/components/message-part.tsx:1103` — adds `dir="auto"` to `<div data-slot="user-message-text">` so user-typed RTL text aligns correctly in the rendered transcript (the `<HighlightedText>` child is span-level so the parent `dir` is what counts).

## Assessment

Smallest possible patch for a real RTL rendering bug. `dir="auto"` is a stable HTML attribute (Baseline since 2017), only inspects the first strong character per element, and is a no-op on pure-LTR content — zero regression surface. Three placements are exactly the right ones (input + assistant body + user body). One nit: any existing CSS that uses `text-align: left` instead of `text-align: start` on descendants will undo the effect — worth a one-line comment, but not blocking. Author tested both Arabic and English in the input.

verdict: merge-as-is
