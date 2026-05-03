# Review: charmbracelet/crush #2652 — fix(grep): stop regex fallback after cancellation

- PR: https://github.com/charmbracelet/crush/pull/2652
- Head SHA: `51423563a79cc81e99afa4e6561f7892c9132163`
- Author: lawrence3699
- Size: +83 / -16 (closes #1855)

## Summary

The grep tool has two implementations: an `rg` (ripgrep) primary path
and a Go-native regex fallback used when `rg` is unavailable or
errors. The fallback (`searchFilesWithRegex`) previously took no
`context.Context`, so a caller-side `ctx.Done()` (timeout or
cancellation from the agent) would never propagate into the
`filepath.Walk`/per-file scan — the fallback would keep walking the
tree and reading files until it finished. This PR threads `ctx`
through the entire fallback path and adds early-exit checks at each
boundary: function entry, after the ripgrep failure, inside the
`Walk` callback (per directory entry), inside `fileContainsPattern`
(per scanned line), and after `Walk` returns. Two regression tests
cover (a) an already-canceled context and (b) a deadline that
expires mid-scan against a 2M-line file.

## Specific citations

- `internal/agent/tools/grep.go:172-184` (`searchFiles` entry) — adds
  an upfront `ctx.Err()` check, then *after* the ripgrep attempt
  fails, re-checks `ctx.Err()` before kicking off the regex fallback.
  This second check matters: if `rg` failed *because* of cancellation
  (e.g. the OS killed the process), there's no point dispatching the
  Go fallback.
- `internal/agent/tools/grep.go:269-272` —
  `searchFilesWithRegex(ctx context.Context, ...)` signature change.
  Adds `ctx.Err()` at function entry.
- `internal/agent/tools/grep.go:298-304` — inside the `filepath.Walk`
  callback. After the per-entry stat, checks `ctx.Err()` and returns
  it (which causes `Walk` itself to terminate). Correct usage of
  `Walk`'s "non-nil error from the callback stops the walk" contract.
- `internal/agent/tools/grep.go:325-330` — after the per-file
  `fileContainsPattern` call, if it returned an error, the code now
  re-checks `ctx.Err()` before the existing "skip files we can't
  read" branch. Important: previously *any* error (including
  `context.Canceled` propagated up from a partial read) was silently
  swallowed as "skip this file". Now cancellation correctly bubbles
  out of `Walk`.
- `internal/agent/tools/grep.go:352-356` — final `ctx.Err()` after
  `Walk` returns. Defends against `Walk` swallowing the cancellation
  somehow (it shouldn't, but cheap insurance).
- `internal/agent/tools/grep.go:378-393` (`fileContainsPattern`) —
  `ctx` threaded to the per-line scanner. Inside the
  `scanner.Scan()` loop, checks `ctx.Err()` per line. This is the
  critical inner cancellation point — for a single 2GB file with no
  matches, the previous code would happily scan the entire file
  before returning, completely independent of any deadline. After
  this change, the deadline is respected at line granularity.
- `internal/agent/tools/grep_test.go:181-198` —
  `TestSearchFilesWithRegex_CanceledContext`: 200 files, ctx
  pre-cancelled, asserts `errors.Is(err, context.Canceled)`. Direct
  coverage of the entry-point check.
- `internal/agent/tools/grep_test.go:200-212` —
  `TestSearchFilesWithRegex_DeadlineExceededDuringScan`: one
  ~80MB file, 5ms deadline, asserts `errors.Is(err,
  context.DeadlineExceeded)`. Direct coverage of the inner-loop
  check. The 5ms window is tight but the file is sized so even a
  fast disk can't finish in time, which is the right test design.

## Verdict

**merge-as-is**

## Rationale

This is exactly the right shape for a context-cancellation bug fix:
thread `ctx` through the entire call stack, add an `ctx.Err()` check
at each natural boundary, and add tests that fail before the patch
and pass after. The check granularity is well chosen — per-line in
the inner scan loop, per-entry in the `Walk` callback, at function
entries, and once after `Walk` returns. Going finer (per-byte)
would cost more than it saves; going coarser (only at function
entry) wouldn't help mid-scan.

The test signatures get a single small refactor — the test helper
maps changed from `func(pattern, path, include string)` to
`func(context.Context, string, string, string)` — and three call
sites updated to pass `t.Context()`. That ripple is unavoidable and
correctly applied.

One subtle correctness point worth highlighting (and getting right):
at `grep.go:325`, the previous code treated any error from
`fileContainsPattern` as "skip this file". The new code re-checks
`ctx.Err()` *before* the skip, so cancellation is no longer
swallowed. Without this re-check, the inner-loop `ctx.Err()` would
propagate up to `fileContainsPattern`'s caller, get classified as
"file we can't read", and silently move on to the next file —
defeating the entire fix. The PR catches this and gets it right.

The `Walk` callback at line 300 returns `err` directly when
`ctx.Err()` is non-nil, which stops the walk. `filepath.Walk`'s
`WalkFunc` contract says any non-nil non-`SkipDir` return halts the
walk and surfaces that error from `Walk` itself — so
`searchFilesWithRegex` then sees that error and propagates it.
Confirmed correct.

No nits. The fix is the right size, the tests are the right tests,
and the diff is mechanical and complete. Ship it.
