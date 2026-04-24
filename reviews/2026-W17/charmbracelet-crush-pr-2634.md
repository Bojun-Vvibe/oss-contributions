# charmbracelet/crush #2634 — fix(logs): guard against unsafe type assertions in printLogLine

**Link:** https://github.com/charmbracelet/crush/pull/2634
**Tag:** silent-default → fail-loud-becomes-fail-soft, defensive-decoding

## What it does

Two bare type assertions in `internal/cmd/logs.go` could panic
`crush logs` on log entries with unexpected shape:

- `int(source["line"].(float64))` panics when a `source` object is
  present but has no `line`, or `line` is not a number.
- `data["time"].(string)` panics when the entry has no `time` field.

Both are replaced with the comma-ok form:

```go
line, ok := source["line"].(float64)
if !ok { continue }
sourceFile := fmt.Sprintf("%s:%d", source["file"], int(line))
```

and

```go
timeStr, ok := data["time"].(string)
if !ok { return time.Now() }
t, err := time.Parse(time.RFC3339, timeStr)
```

The fallback semantics already existed for adjacent failure modes
(skip-source, fall-back-to-time.Now on parse error), so the new branches
slot into existing policy.

## What it gets right

- **Smallest possible diff** for the bug. Two assertions, two ok-checks,
  no churn elsewhere. Trivial to review and trivial to backport.
- Re-uses the **existing** policy on each branch: the `source` block
  was already a `continue`-on-bad-shape loop iteration, and the
  `time.Parse` error already returned `time.Now()` — the new ok-checks
  just extend that policy to the assertion-typed inputs.
- Doesn't introduce a new helper, a new error type, or a new logging
  call. Behavior is purely "don't panic on malformed log lines."

## Concerns / risks

- **Silent-default trade-off.** Both new branches swallow malformed
  data with no log/metric. For a CLI command (`crush logs`) that
  literally exists to surface logs, silently dropping
  `source.line == "abc"` or `time == 12345` (number, not RFC3339
  string) is the wrong default — these are caller bugs in the log
  producer that the user wants to know about. A
  `fmt.Fprintf(os.Stderr, "skipping malformed log entry: …")` would
  preserve the no-panic guarantee while keeping the diagnostic
  value. Pure fail-soft makes the producer bug invisible.
- The `time` fallback to `time.Now()` is the existing pattern but it's
  worth questioning: a log entry with no timestamp printed at
  wall-clock-now is a **misleading reordering** in the output stream,
  not a missing-data signal. A sentinel zero-time + visible marker
  would be more honest.
- No regression test. A four-line table-test against `printLogLine`
  with three malformed inputs (no `time`, no `source.line`,
  `source.line` as string) would lock the no-panic guarantee against
  future refactors that re-introduce bare assertions.
- The PR body's "Test plan" still has unchecked boxes. `go build` is
  not a behavior check.

## Suggestion

Add a `t.Run("malformed", …)` table test exercising the three panic
shapes the assertion previously hit, plus optionally surface a
`stderr` warn line on each soft-skip so producer-side bugs don't go
silent. The fix is correct as-is for "stop panicking"; the gap is
"now we fail without telling anyone."
