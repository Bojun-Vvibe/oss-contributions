# sst/opencode PR #25866 — fix(ui): preserve SVG tags in DOMPurify config for KaTeX math rendering

- URL: https://github.com/sst/opencode/pull/25866
- Head SHA: `410fbad5b5c4259ccf5014ca0d264b40269a4ffc`
- Author: zharinov
- Closes #25812
- Size: +2 / -0 (one file)

## Verdict
`merge-as-is`

## Rationale

The fix at `packages/ui/src/components/markdown.tsx:36-37` is the minimal correct change for the bug it
targets: KaTeX renders `\sqrt`, `\overrightarrow`, `\widehat`, `\widetilde`, `\overbrace`, and
`\underbrace` as inline `<svg><path d="…"/></svg>` markup, and the existing DOMPurify config —
which keeps `SANITIZE_NAMED_PROPS: true`, `FORBID_TAGS: ["style"]`, and
`FORBID_CONTENTS: ["style", "script"]` — strips any tag not in the default allow-list. Adding
`ADD_TAGS: ["svg", "path"]` plus `ADD_ATTR: ["d", "viewBox", "preserveAspectRatio", "xmlns"]` is
exactly the four-attribute set KaTeX actually emits on these primitives, and the PR does not
loosen anything else (no `foreignObject`, no `script`, no `on*` handlers — DOMPurify continues to
strip event handlers on SVG elements per its default deny list).

The PR body shows hand-verification across all six affected KaTeX primitives plus radical-with-index
(`\sqrt[4]{...}`, `\sqrt[7]{...}`) which is the case most likely to regress because it's actually two
nested SVG nodes; the author confirmed all render. Scope is tight (single file, two added lines, no
test deletions, no behavior changes outside the markdown sanitizer), the surface is the right one
(server-side / desktop renderer of LLM output where math is common), and the alternative (rendering
KaTeX to MathML and dropping the SVG fallback) would be a multi-PR refactor.

The one residual risk worth naming for the merger but not blocking: `xmlns` on inline SVG inside
HTML5 is technically inert (the parser already places it in the SVG namespace), so it's defensive
not load-bearing — fine to keep, just noting it isn't required for KaTeX correctness. No tests are
added but the existing `markdown.tsx` surface has no unit tests for DOMPurify config and adding one
just for this allow-list would be disproportionate to a 2-line fix.
