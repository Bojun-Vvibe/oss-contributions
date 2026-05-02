# Review: charmbracelet/crush #2779

- **PR:** charmbracelet/crush#2779 — `fix(tools/view): honor limit parameter on files exceeding MaxViewSize`
- **Head SHA:** `65b74cd5cdfec06892aa6dde5a44fca2d74fd44a`
- **Files:** 1 (+2/-1)
- **Verdict:** `merge-after-nits`

## Observations

1. **Fix is exactly right** — `internal/agent/tools/view.go:165` previously rejected any file `> MaxViewSize` (100 KB) before consulting `params.Limit`. The change adds `&& params.Limit <= 0` so the size guard only fires when the caller did not specify a bounded read. This matches how every other paginated read tool in the codebase behaves.

2. **No upper bound on `params.Limit`** — Now that limit bypasses the size guard, a malicious or buggy agent passing `limit=1000000000` would attempt to read all of a multi-GB file. `readTextFile` presumably handles line-bounded reads efficiently, but it's worth confirming there's an upstream cap on `limit` (probably in the tool param schema). If not, a `min(params.Limit, MaxLines)` clamp here is a one-liner.

3. **Missing offset behavior** — The PR only changes the size guard; it does not address the same class of bug for `params.Offset`. If a caller passes `offset=50000` on a 200KB file (intending to skip past the size guard area), the same error will fire with `limit <= 0`. Probably out of scope here, but worth noting in the issue.

4. **No new test** — The author acknowledges this and points to manual repro. The existing `view_test.go` is presumably small; adding a `TestView_LargeFileWithLimit` case (create file > MaxViewSize, call with `limit=20`, assert success and 20 lines returned) is ~15 lines and locks in the fix. Without it, a future refactor that reorders the guards will re-introduce the regression silently.

## Recommendation

Land after a regression test is added; the one-line fix itself is clearly correct.
