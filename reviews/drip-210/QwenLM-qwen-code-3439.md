# QwenLM/qwen-code#3439 — feat(cli): render LaTeX math in markdown output

- PR: https://github.com/QwenLM/qwen-code/pull/3439
- Head SHA: `ffa55aba3f4d878a177ea6d0ace5386ce0c34478`
- Author: reidliu41
- Files: 7 changed, +1507/−129. New file
  `packages/cli/src/ui/utils/TerminalMathRenderer.ts` is the bulk
  (+1061).
- Closes #2620

## Context

Math-heavy assistant responses currently render as raw `$E = mc^2$` or
`$$\frac{a}{b}$$` strings in the terminal — readable but ugly. The CLI
already has a custom markdown renderer (`MarkdownDisplay.tsx`,
`InlineMarkdownRenderer.tsx`, `TableRenderer.tsx`); this PR plumbs in a
new `TerminalMathRenderer` module and threads inline + block math through
the existing render passes.

## Design

The change has three layers.

**New module `TerminalMathRenderer.ts`** (+1061). Three exports referenced
from the rest of the diff:

- `splitInlineMathSegments(text)` — returns
  `Array<{type: 'text' | 'math', text: string}>`. Used at
  `InlineMarkdownRenderer.tsx:37` to fast-path `text` containing no math
  markers (`$...$` / `\(...\)`).
- `renderTerminalMathInline(s)` — converts a single inline math segment
  into Unicode (`E = mc²`, `α + β ≤ γ`, superscript/subscript digits).
- `renderTerminalMathBlock(...)` — used inside `RenderMathBlock` (referenced
  at `MarkdownDisplay.tsx:171`) to produce stacked fractions, `cases`,
  `pmatrix`/`bmatrix`/etc., and large operators with limits.

**Inline integration** (`InlineMarkdownRenderer.tsx:37-65`). The renderer
checks `mathSegments.some(s => s.type === 'math')`; if any math segment is
present, it emits a `<>` fragment that maps each segment to either
`<Text color={theme.text.code}>{renderTerminalMathInline(...)}</Text>` (for
math) or recursively renders the markdown nodes for text segments. Crucially,
the recursion is split out into a pure helper `renderInlineMarkdownNodes(text,
textColor, keyPrefix)` (`:69-205`) so the no-math path is unchanged
character-for-character (`:73-78`):

```ts
if (!/[*_~`<[https?:]/.test(text)) {
  return [<Text key={`${keyPrefix}-plain`} color={textColor}>{text}</Text>];
}
```

This is a pure refactor of the previous body — same regex, same node-
emission logic, just hoisted into a function so the math wrapper can call
it with a per-segment `keyPrefix`. Spot-checked the regex
`/[*_~`<[https?:]/` is byte-identical to the previous version.

**Block integration** (`MarkdownDisplay.tsx:75-200`). Adds an `inMathBlock`
state machine alongside the existing `inCodeBlock` / `inTable` machines.
The opening regex is `/^ *(\$\$|\\\[)(.*)$/` (`:49`) — leading whitespace
allowed, two delimiter forms (`$$` and `\[`). `getMathBlockStart` returns
`{content, close: '$$' | '\\]', isClosed}` — `isClosed` covers the
single-line case `$$ x^2 $$` (`:96-103`). Inside the block, the code looks
for `mathBlockClose` and on match emits `<RenderMathBlock>` then continues
parsing any trailing text (`:148-170`).

**Width calculation hook** (`InlineMarkdownRenderer.tsx:213`). The existing
`getPlainTextLength` is updated to call `renderInlineMathInText(text)` *before*
the strip-markdown pipeline. This is the single most important non-obvious
change — the table-column-width calculator (`TableRenderer.tsx`, +15) needs
to measure the *rendered* math width, not the raw `$...$` width, otherwise
columns get sized for the LaTeX source instead of the rendered Unicode.
Without this, math inside a markdown table would either truncate or leave
huge whitespace gaps.

## What the test suite proves

`MarkdownDisplay.test.tsx:38-95` — five new tests, each pinning a non-obvious
behavior:

- **`renders inline math without rewriting non-math dollar text`** (`:38-44`).
  Input `'Energy $E = mc^2$ costs $20 and $30.'`. Output must contain
  `'Energy E = mc² costs $20 and $30.'`. This is the test that catches the
  classic over-eager-regex bug: a naive `\$([^$]+)\$` would match
  `$20 and $30` as a math span. The actual `splitInlineMathSegments` must
  reject single-side dollar usage.
- **`renders block math with terminal layout`** (`:46-55`). Input
  `'$$\n\\frac{a+b}{c+d}\n$$'`. Output must contain `'a+b'`, `'───'`, and
  `'c+d'` — i.e., the box-drawing fraction bar is actually emitted.
- **`does not render math inside fenced code blocks`** (`:57-63`). A
  fenced ```latex code block with `$E = mc^2$` inside must come out as
  literal `$E = mc^2$`. The test pins that the math state machine never
  enters while `inCodeBlock` is true.
- **The 30-line "manual math verification" test** (`:65-95`) is the
  integration smoke test — mixed inline, block, `cases`, an unclosed
  ```latex fence, and trailing text with `$20` and `$PATH` literals. This
  is the test that exercises the full state-machine interaction.

## Risks / nits

- **+1061 lines for `TerminalMathRenderer.ts` is a lot.** The PR
  description is explicit that this is "not a full TeX engine" and the
  fallback for unknown LaTeX is to preserve the source text — which is the
  right policy. But a 1k-line file with a custom mini-parser deserves
  either (a) an integration with an existing library like `unicodeit` or
  `mathjax-cli` rendering, or (b) a clear API boundary file at the top
  documenting the public exports and the "preserve unknown" contract. The
  diff window doesn't show the file's structure, so this is a nudge, not
  a blocker.
- **`inMathBlock` state is global to the parse pass** but doesn't appear
  to interact with `inTable` — what happens if a table row contains
  ` |$$\frac{a}{b}$$| ` ? Inline math inside tables is handled (per the
  width-calc hook above), but block-math-as-table-cell is not addressed.
  Likely out of scope; worth a TODO.
- **`renderTerminalMathInline` colors output `theme.text.code`** at
  `InlineMarkdownRenderer.tsx:43`. If a user's theme makes `text.code`
  visually identical to `text.primary`, math becomes invisible-to-distinguish
  from prose. Consider a separate `theme.text.math` semantic color.

## Verdict

**`needs-discussion`**

The feature is valuable (math-heavy responses are a real pain point), the
state-machine integration with `MarkdownDisplay`/`InlineMarkdownRenderer`/
`TableRenderer` is principled, the test suite catches the classic
over-eager-regex bug, and the no-math fast path is preserved
character-for-character. But the +1061-line in-tree mini-parser is a
maintenance commitment that deserves a sit-down before merge: is this the
right scope, or should the renderer be a thin shim over an existing
unicode-math library? The `theme.text.math` color question is also a
design call worth making before users get used to the current shade.

If the maintainers are willing to own the in-tree parser, this is a
`merge-after-nits` (split `TerminalMathRenderer.ts` into
`tokenizer/parser/renderer` files, add the `theme.text.math` color, add
the block-math-in-table-cell TODO). If they'd prefer a library, this
should be re-architected before merge.

## What I learned

The single most important change in this PR is the four-line edit to
`getPlainTextLength` — without it, every other change would visually look
right in isolation and silently break tables. This is the pattern for
adding a new inline-rendering pass to any markdown system: find the
width-measurement function and update it *first*, before the rendering
itself. The test that pins "non-math `$20` survives" (`:38-44`) is the
canonical test for any inline-math feature — it's the bug that ships if
you skip it.
