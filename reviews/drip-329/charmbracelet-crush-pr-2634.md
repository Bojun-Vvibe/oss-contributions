# charmbracelet/crush #2634 — fix(logs): guard against unsafe type assertions in printLogLine

- SHA: `ed5b5dd523474570ee904967bdddf050d20714fc`
- State: OPEN, +10/-2 across 1 file
- Files: `internal/cmd/logs.go`

## Summary

Two bare type assertions in `printLogLine` (`source["line"].(float64)` and `data["time"].(string)`) panic when fields are missing or wrong type. PR replaces both with the comma-ok form. Fallback semantics preserved (skip source field / fall back to `time.Now()`).

## Notes

- `internal/cmd/logs.go:189-193` — `source["line"]` now ok-checked. On miss, `continue` skips appending the `source` key/value pair to `otherData`. Matches the existing fallback path in the surrounding `case "source":` arm where a `continue` already handles a non-map `source`. Consistent.
- `internal/cmd/logs.go:202-205` — `data["time"]` now ok-checked. On miss, the `SetTimeFunction` callback returns `time.Now()` directly, bypassing the `time.Parse` call. This matches the inline comment two lines below (`// fallback to current time if parsing fails`) which previously only handled parse error, not missing field. Good intent alignment.
- The `int(source["line"].(float64))` cast is preserved exactly — JSON unmarshals all numbers to `float64`, so the post-ok-check `int(line)` is the right truncation. Behavior preserved when the value is present.
- Both fixes are a strict superset of previous behavior: previously valid inputs still produce identical output; previously panicking inputs now degrade gracefully.
- No test added. The PR's own test plan checkboxes are unchecked: `[ ] go build ./...` and `[ ] Run crush logs`. For a panic-fix this small, an inline test with a `printLogLine` call that feeds malformed JSON would be ~15 LOC and would lock the regression. Not a blocker but cheap to add.
- Reproduction described in PR body is plausible: any log line emitted by a logger that omits `time` (some structured loggers default-strip when value equals zero) or any `source` map without `line` (e.g., Go `slog` source records when `AddSource: false` partially-populates) would panic on master.
- Generated-with footer mentions an editor extension; no concern with the change itself.

## Verdict

`merge-after-nits` — correct and minimal. Tick the test-plan boxes and ideally add a 10-line table-driven test for `printLogLine` with three cases (happy path, missing `line`, missing `time`).
