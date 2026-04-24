# PR-2652 — fix(grep): stop regex fallback after cancellation

[charmbracelet/crush#2652](https://github.com/charmbracelet/crush/pull/2652)

## Context

`internal/agent/tools/grep.go` — `searchFiles` runs ripgrep first, and
if `searchWithRipgrep` returns an error, falls back to a pure-Go
`searchFilesWithRegex` walk. That fallback was completely
context-unaware: it used `filepath.Walk` + a buffered scanner over each
text file, with zero `ctx.Err()` checks. If the agent cancelled the
tool call (user hit Esc, deadline expired, parent context cancelled),
ripgrep would honor the cancel — but if ripgrep had already failed for
some unrelated reason, the regex walk kept going to completion across
potentially millions of lines.

The fix threads `context.Context` through the entire fallback path:

1. `searchFilesWithRegex(ctx, pattern, rootPath, include)` — was
   `(pattern, rootPath, include)`.
2. `fileContainsPattern(ctx, filePath, pattern)` — was
   `(filePath, pattern)`.
3. Adds `ctx.Err()` short-circuits at every meaningful boundary:
   entry to each function, inside the `filepath.Walk` callback before
   each file decision, before each call to `fileContainsPattern`,
   inside the per-line `scanner.Scan()` loop, and after the walk
   returns.
4. Inside `searchFiles`, after `searchWithRipgrep` errors, also checks
   `ctx.Err()` *before* falling through to the regex path — so a
   cancellation that happens to surface as an `rg` error doesn't get
   masked into a full Go-side rescan.

Two new tests:

- `TestSearchFilesWithRegex_CanceledContext` creates 200 nested files,
  cancels the context, asserts `ErrorIs(err, context.Canceled)`.
- `TestSearchFilesWithRegex_DeadlineExceededDuringScan` writes a
  ~76 MB file (2M lines), sets a 5ms deadline, asserts
  `ErrorIs(err, context.DeadlineExceeded)`.

The existing test tables (`TestGrepWithIgnoreFiles`,
`TestSearchImplementations`) get their function signatures updated to
the new `(context.Context, ...)` shape.

## Why it matters

Long-running, un-cancellable search is the worst kind of resource leak
in an interactive agent: the user has *visibly* moved on (Ctrl-C'd,
typed a new prompt, switched session), but the goroutine is pinned on
disk I/O for the next several seconds and any results it returns will
be discarded by the caller. CPU and disk are wasted; on slow disks
this also blocks subsequent tool invocations.

## Strengths

- Cancellation discipline is *layered* the right way — every level of
  the call stack re-checks `ctx.Err()` rather than relying on a single
  upstream check. That's the correct posture because a single check
  can be invalidated by any blocking call below it.
- The check inside the `scanner.Scan()` loop is the most important
  one: a single 76MB file would otherwise pin the goroutine for ~100ms+
  with no escape. The deadline test specifically exercises that path
  with a 5ms deadline against a 2M-line file.
- The check *before* falling through to regex (after `rg` errored) is
  subtle but correct: if ripgrep died because its child context was
  cancelled, we shouldn't immediately re-do the search in pure Go.
- Test signature updates are propagated cleanly to both existing
  table-driven tests; same `t.Context()` is used for both `regex` and
  `rg` lambdas, keeping behavior parity.
- `require.ErrorIs(err, context.Canceled)` (not `assert.Equal`) is
  the right matcher — survives any wrapping.

## Concerns / risks

- The `filepath.Walk` callback returns `err` instead of `nil` when
  `ctx.Err()` fires, but in another branch returns `nil // Skip files
  we can't read` even *after* checking `ctx.Err()` returns non-nil.
  Quick read of the diff:

  ```go
  match, lineNum, charNum, lineText, err := fileContainsPattern(ctx, path, regex)
  if err != nil {
      if err := ctx.Err(); err != nil {
          return err
      }
      return nil // Skip files we can't read
  }
  ```

  Good — that's correct. The propagation only happens when the error
  is *due* to cancellation. Worth a comment explaining the precedence,
  though.
- `bufio.NewScanner` defaults to a 64KB token limit. Files with lines
  longer than 64KB will cause scanner.Err() to surface
  `bufio.ErrTooLong` — which the new code path wraps cleanly, but
  there's no test for "large single-line file + cancellation" race.
- The deadline test uses 5ms which is *very* tight on a slow CI runner
  (especially under load). If the file write + initial walk takes >5ms
  before the first scanner.Scan, the test could pass via the early
  `ctx.Err()` check rather than the per-line check it intends to
  exercise. A longer deadline + slower file (e.g. sleep injection in a
  fake reader) would isolate the loop check.
- `searchWithRipgrep` is unchanged. If `rg` itself is the offending
  long-runner that doesn't honor SIGTERM cleanly, this PR doesn't help
  that path. But the diff doesn't claim to.

## Suggested follow-ups

- Add a test where a single file >64KB-per-line is searched under
  cancellation, to lock down the `bufio.ErrTooLong` interaction.
- Consider wrapping the `filepath.Walk` in a custom walker that yields
  control more frequently for very deep trees (Walk holds a single
  goroutine).
- Document the precedence rule in `fileContainsPattern`: "if both
  ctx.Err() and a read error are present, ctx.Err() wins".
- Add a metric/log at the `searchFiles` level for how often the regex
  fallback fires and how often it gets cancelled — useful for tuning
  whether to keep the fallback at all.
