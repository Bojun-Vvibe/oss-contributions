# charmbracelet/crush PR #2656 — fix: use same chroma formatter as diffview for markdown

- **Repo:** charmbracelet/crush
- **PR:** [#2656](https://github.com/charmbracelet/crush/pull/2656)
- **Head SHA:** `383721f372cb700d1c0f36bd0f4230fd492a0480`
- **Size:** +76/-60 across 3 files (all in `internal/ui/`)
- **Status:** merged
- **Reviewer:** Bojun (drip-23)

## Summary

Refactor that extracts the `chromaFormatter` previously embedded in
`internal/ui/diffview/chroma.go` into a shared `internal/ui/xchroma/
chroma.go` package, and wires the markdown renderer
(`internal/ui/common/markdown.go`) to use the same formatter via
glamour's `WithChromaFormatter` registration. The motivating bug:
markdown code blocks used a different chroma formatter than diff
panels, so syntax highlighting in markdown was visually inconsistent
with diffs of the same language.

Net behavior change is small (markdown code blocks now render with
the same Lip Gloss-based foreground + forced-background color
scheme as the diff view), but the refactor is the bulk of the diff.

## What's changed

### 1. New shared package `internal/ui/xchroma/chroma.go` (+52 lines)

Diff lines 135–192. The new package exports a single function:

```go
func Formatter(bgColor color.Color, processValue func(string) string) chroma.Formatter {
    return chroma.FormatterFunc(func(w io.Writer, style *chroma.Style, it chroma.Iterator) error {
        for token := it(); token != chroma.EOF; token = it() {
            value := token.Value
            if processValue != nil {
                value = processValue(value)
            }
            ...
```

Two design decisions worth calling out:

- **`processValue` is a callback parameter, not baked in.** The old
  `chromaFormatter` (deleted file `internal/ui/diffview/chroma.go`,
  diff lines 44–106) hard-coded `value = strings.TrimRight(value,
  "\n")` and `value = ansiext.Escape(value)` on every token. The new
  shared formatter makes that processing optional — diffview passes
  the same logic via `processChromaValue` (diff lines 130–134), but
  markdown rendering passes `nil` (diff line 23: `formatters.Register
  (formatterName, xchroma.Formatter(zero, nil))`). That's deliberate
  and correct: markdown code blocks shouldn't strip trailing
  newlines (it would collapse blank lines inside code) and don't
  need the diff-specific ANSI-escape pass.

- **`chroma.FormatterFunc` adapter** instead of a struct implementing
  `chroma.Formatter`. The old code used a struct (`type chromaFormatter
  struct { bgColor color.Color }`) with a `Format` method. The new
  shape is a closure inside `FormatterFunc`. Both compile to the
  same interface, but the closure shape lets `bgColor` and
  `processValue` close over the parameter scope without a struct.
  This is fine; minor style preference.

### 2. `internal/ui/common/markdown.go` (+13/-0)

Diff lines 1–43. Adds a package-init that registers the new formatter
under the name `"crush"` with chroma's global formatter registry:

```go
const formatterName = "crush"

func init() {
    var zero color.Color
    formatters.Register(formatterName, xchroma.Formatter(zero, nil))
}
```

Then both `MarkdownRenderer` (diff lines 28–33) and
`PlainMarkdownRenderer` (diff lines 36–42) gain a
`glamour.WithChromaFormatter(formatterName)` option to opt their
TermRenderer into the registered formatter.

The comment at diff line 21–22 ("Glamour does not offer us an option
to pass the formatter implementation directly. We need to register and
use by name.") is the right thing to flag — that's a glamour API
limitation forcing the global-registry workaround.

### 3. `internal/ui/diffview/diffview.go` (+8/-3)

Diff lines 107–134. The diffview's `getChromaFormatter` method now
delegates:

```go
func (dv *DiffView) getChromaFormatter(bgColor color.Color) chroma.Formatter {
    return xchroma.Formatter(bgColor, processChromaValue)
}
```

with the trailing-newline-strip + ansiext.Escape logic moved into a
new `processChromaValue` helper (diff lines 130–134). The old
`chromaFormatter` struct file is deleted entirely (diff lines 44–106).

## Concerns

1. **Global-registry side effect at package import**

   `init()` at diff line 19 of `markdown.go` registers `"crush"` in
   chroma's global formatter table. This is *fine* in production
   (single binary, single registration) but creates two latent
   issues:

   - **Test-binary import order.** If tests import
     `internal/ui/common` more than once (or in a way that triggers
     re-registration), `formatters.Register` semantics matter — does
     it overwrite, append, or error? Worth a one-line comment in the
     init function noting the assumption (and ideally a test that
     imports the package twice).
   - **Name collision.** The string `"crush"` is short enough that a
     downstream consumer (or a chroma plugin) could collide. A
     namespaced `"crush.markdown"` or
     `"crush-internal-formatter"` would be safer.

2. **`var zero color.Color` is `nil`-ish**

   Diff line 22 uses Go's zero value for `color.Color` (which is the
   nil interface). The downstream `lipgloss.NewStyle().Background
   (bgColor)` call at diff line 170 of `xchroma/chroma.go` will be
   passed `nil` for the markdown path. `lipgloss.Style.Background`
   with a `nil` `color.Color` is — in current lipgloss — a no-op
   (background is unset rather than rendered as the zero color). That
   matches the intent (markdown code blocks shouldn't have a forced
   background, only the syntax foreground colors should be applied),
   but it's relying on undocumented lipgloss behavior. Worth either
   documenting the assumption inline ("nil bgColor disables forced
   background") or making `bgColor` an `*color.Color` (or
   `chroma-style-background` enum) so the intent is explicit.

3. **No test coverage for the markdown path**

   Diffview already has visual snapshot tests; the new markdown
   formatter wiring has no test asserting that
   `MarkdownRenderer` produces output containing the foreground
   colors from the registered formatter. A simple golden test
   feeding a fenced code block (` ```go\nfunc f() {}\n``` `) and
   asserting the output contains an ANSI SGR sequence would catch
   future regressions where chroma's `WithChromaFormatter` API
   changes name or ordering.

4. **`processChromaValue` could be a method on `DiffView`**

   The new free function in diffview.go (diff lines 130–134) only
   makes sense in the diffview context. Making it
   `(dv *DiffView) processChromaValue` is a one-line change that
   keeps the API surface tight and matches the surrounding
   methods. Minor style nit.

5. **Dead-code after removal**

   The deleted `internal/ui/diffview/chroma.go` (diff lines 44–106)
   removed the `chromaFormatter` struct cleanly, and the `_ chroma.
   Formatter = chromaFormatter{}` interface assertion went with it.
   The new shared formatter has no equivalent compile-time interface
   assertion. Adding `var _ chroma.Formatter = (chroma.FormatterFunc)
   (nil)` somewhere isn't strictly necessary (FormatterFunc already
   implements the interface) but the original code had it for a
   reason — to fail at compile time if the chroma interface changes.

## Verdict

`merge-as-is` (since this is already merged) — would have been
`merge-after-nits` at review time:

- namespace the formatter registration name (`crush` → something
  collision-resistant);
- inline-document the `nil bgColor` → "no forced background"
  assumption;
- one golden test for the markdown code-block path.

The refactor itself is a clean DRY win — the old code had two
chroma formatters that shared 90% of their body, differing only in
trailing-newline-strip and ansiext.Escape. Pulling those two
divergences out as a `processValue` callback is exactly the right
factoring.

## What I learned

The "shared formatter, parameterized post-processing" pattern here
is the right way to reuse chroma styling across multiple TUI
surfaces. The lurking complication is glamour's API: it accepts a
formatter *name* (lookup in chroma's global registry) rather than a
formatter *instance*, which forces every consumer to do
package-init-time registration. That's a minor framework smell that
crush is working around well, but it's worth tracking — if
glamour ever exposes `WithChromaFormatterFunc(chroma.Formatter)`,
the `init()` registration here can collapse into a single direct
parameter pass and the global-registry concerns from #1 above go
away entirely.
