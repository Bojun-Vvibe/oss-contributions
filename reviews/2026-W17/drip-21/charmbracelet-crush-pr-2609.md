# charmbracelet/crush#2609 — feat(session): add `session export` command for markdown/JSON export

- **PR:** https://github.com/charmbracelet/crush/pull/2609
- **Head SHA:** `e472fff4d22632aa7b3ab675bf6df7728d83a140`
- **Files:** `internal/cmd/session.go` (+182/-15), `internal/cmd/session_markdown_test.go` (+376/-0).
- **Verdict:** **request-changes**

## Context

Closes `#1965`. Users requested a way to export sessions as
self-contained markdown documents for sharing/archival. The PR adds
markdown export logic plus a comprehensive test suite.

## Title-vs-implementation mismatch

The PR is titled and described as adding a *new* `session export`
subcommand:

> *"Adds a `crush session export <id>` subcommand that exports a
> session as a self-contained markdown document"*
>
> Usage examples in the body:
> ```
> crush session export abc1234
> crush session export abc1234 > session.md
> crush session export abc1234 --json
> ```

But the diff does **not** add a new `sessionExportCmd`. It instead
extends the existing `session show` and `session last` subcommands
with `--markdown` / `-o <file>` flags. The actual user-facing
invocation is:

```
crush session show abc1234 --markdown
crush session show abc1234 -o session.md
```

The PR's own test file is named `session_markdown_test.go`, which
is consistent with what the diff actually does, not with what the
PR title and body promise. This is a **substantive disconnect**:
issue #1965 asked for "export"; the PR ships "show with format
flags". Reviewers and future readers searching for
`crush session export` will find nothing.

The maintainers need to decide:

- (a) Rename the PR to *"feat(session): add `--markdown` / `-o` to
  `session show` and `session last`"* and accept that the
  subcommand isn't materializing, or
- (b) Refactor to actually add `sessionExportCmd` and have it
  delegate to the same renderer.

Either is fine but the current state is incoherent.

## Design — what's actually added

### Format-resolution helper (`session.go:357-377`)

```go
func sessionOutputFormat(jsonFlag, mdFlag bool, outputPath string) string {
    if jsonFlag { return "json" }
    if mdFlag { return "markdown" }
    if outputPath != "" {
        switch strings.ToLower(filepath.Ext(outputPath)) {
        case ".md", ".markdown": return "markdown"
        case ".json":            return "json"
        }
    }
    if !term.IsTerminal(os.Stdout.Fd()) { return "markdown" }
    return "human"
}
```

Sensible precedence (`flags > extension > tty > human`). Two
concerns:

- **The non-TTY default flips from "human" to "markdown".** Anyone
  piping `crush session show <id> | grep ...` today gets
  human-formatted output (with terminal escapes stripped or
  preserved depending on the renderer); after this PR they get
  markdown. That's a behavior change for an existing command,
  unannounced in the PR body. Existing scripts that grep for
  `Title:` or similar human-format anchors silently break.
- **The `term.IsTerminal(os.Stdout.Fd())` check is correct but
  global.** Tests that don't redirect stdout will get
  TTY-detection-dependent results unless the tests substitute
  `cmd.SetOut(...)`. Looking at the test file structure, this is
  handled, but it's fragile — a future test that omits the
  override gets flaky CI on local-dev vs CI environments.

### Markdown renderer (`session.go:378-471`)

Produces YAML frontmatter (`title`, `id`, `date`, `cost`, `tokens`),
then per-message blocks:

- `## User` — text only.
- `## Assistant` — model attribution, optional `<details>` for
  reasoning/thinking, text body, then `### Tool Call: …` blocks.
- `## Tool` — `### Tool Result: …` with `<details>` collapsible
  output and an `**Error:**` prefix when `tr.IsError`.

System messages skipped (intentionally per PR body).

Quality concerns:

1. **`tc.Input` is dumped raw inside ```` ```json ```` fences with
   no validation** (`session.go:444-447`):
   ```go
   if tc.Input != "" {
       fmt.Fprintln(&buf, "```json")
       fmt.Fprintln(&buf, tc.Input)
       fmt.Fprintln(&buf, "```")
   ```
   If `tc.Input` is malformed JSON, ill-encoded, or contains a
   triple-backtick (which is rare but legal), the output
   markdown is corrupted. At minimum: validate it parses; if not,
   fall back to a `text` fence. Triple-backtick collision can be
   handled by switching to `~~~` fences when the content contains
   ``` ``` ```.
2. **`<details>` tag for reasoning is not GitHub-Flavored-Markdown
   safe in all renderers.** GFM allows it, CommonMark does not. The
   PR body advertises this as "self-contained markdown" — for the
   majority of consumers (gist, GitHub PRs, Obsidian) it works; for
   strict CommonMark renderers (some static-site generators) it
   degrades to inline HTML literal text. Worth a one-line comment
   acknowledging the GFM dependency.
3. **Title sanitization is minimal.** `strings.ReplaceAll(sess.Title,
   "\n", " ")` covers newlines but not other markdown-significant
   characters (`#`, `[`, `]`, `*`, `_`). A title like `[fix] crash
   in *parser*` will render with brackets as a broken link reference
   and asterisks as bold. Either escape with backslash or wrap in
   backticks.
4. **YAML frontmatter is hand-rolled.** `fmt.Fprintf(&buf,
   "title: %q\n", sess.Title)` — using Go's `%q` produces a
   JSON-style escape, not a YAML one. It happens to work for most
   ASCII titles but fails on titles containing `\u` escape
   sequences when the consumer uses a strict YAML parser. Use
   `gopkg.in/yaml.v3` (likely already in deps) for the frontmatter
   block.
5. **`session.HashID(sess.ID)[:12]` panic risk.** If `HashID`
   returns shorter than 12 chars (it shouldn't, but the API
   doesn't guarantee), this is an out-of-bounds slice. Either use
   `sess.ID` directly or guard.

### File-write path (`session.go:339-352`)

```go
if outputPath != "" {
    f, err := os.Create(outputPath)
    if err != nil { return fmt.Errorf("failed to create output file: %w", err) }
    defer f.Close()
    w = f
} else {
    w = cmd.OutOrStdout()
}
```

Two issues:

- **`defer f.Close()` swallows close errors** — for write paths,
  close errors can be the only signal that the OS-level flush
  failed (out of disk, network filesystem hiccup). The renderer
  writes to a `strings.Builder` and then dumps in one
  `io.WriteString` call so most write errors surface synchronously,
  but the close error is the last line of defense.
- **No `os.O_EXCL`.** `os.Create` clobbers an existing file. A
  user typing `crush session show abc -o session.md` and
  realizing they meant `session2.md` after an existing
  `session.md` is silently overwritten. Either honor a `--force`
  flag (and refuse without it) or use `os.OpenFile(path,
  os.O_WRONLY|os.O_CREATE|os.O_EXCL, 0o644)`.

## Strengths

- **376 lines of test coverage** on the markdown renderer alone —
  basic conversation, tool calls with input, reasoning blocks,
  system-message filtering, error tool results, zero-cost/zero-token
  sessions, and the format-helper precedence matrix. This is
  significantly more test rigor than typical for a TUI feature
  PR.
- **Reuses existing `outputSessionJSON`** — the JSON path is not a
  separate implementation, just a flag override on the existing
  formatter.
- **Telemetry hook** — `SessionExported` event tracks format,
  reasonable for understanding adoption.

## Verdict — request-changes

The feature is genuinely useful and the test coverage is excellent.
The blockers:

1. **Resolve the title-vs-implementation mismatch.** Either rename
   or actually ship the subcommand.
2. **Don't silently flip the non-TTY default of `session show`.**
   Either preserve current human-format-when-non-TTY behavior, or
   call out the breaking change explicitly.
3. **Don't clobber existing files via `os.Create`** without a
   `--force` flag.
4. **Tighten the markdown safety**: triple-backtick collision in
   `tc.Input`, title escaping, `HashID` slice bound.

Once those are addressed this is a clean merge.

## What I learned

PRs whose title and body promise a user-facing subcommand, but whose
diff actually adds flags to an existing command, are a class of bug
that gets through review surprisingly often — the body sets reviewer
expectations and we read the diff for *whether the body is true*
rather than *what the diff actually does*. The fix is to read the
flag-registration block (`init()`) before reading the renderer:
that's where the user surface is defined, and any disagreement
between `init()` and the PR body is the reviewer's first signal that
the PR has shipped something other than what it claims. Separately,
silently flipping a default output format when stdout is non-TTY is
a backwards-incompatible change that's almost always more
disruptive than the feature it enables — pipes-and-grep workflows
are the silent majority of CLI usage.
