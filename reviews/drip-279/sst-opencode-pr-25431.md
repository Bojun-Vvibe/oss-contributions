# sst/opencode PR #25431 — tweak: allow read tool to accept offset of 0

- URL: https://github.com/sst/opencode/pull/25431
- Head SHA: `2f20279dad0a668d1632eb1925c17f93ad381392`
- Files touched: 1 (+2 / −6)
- Verdict: **request-changes**

## Summary
Removes the explicit `params.offset < 1` validation in `ReadTool` and switches the
default-coalescing operator from `??` to `||` in two call sites so that an offset
of `0` no longer errors and instead falls back to `1`. Intent (per title) is to
let agents pass `offset: 0` without tripping the validator.

## Cited concerns

1. `packages/opencode/src/tool/read.ts:191` (post-patch):
   `const offset = params.offset || 1` — using `||` instead of `??` silently
   coerces `0` to `1`. That means a caller that explicitly asks for "start at
   index 0" still gets index 1 behavior. If the goal is "0 should work", this
   only papers over the validator removal; it doesn't actually honor `0`. The
   schema/comment elsewhere in the file documents 1-based offsets, so the real
   fix is either (a) keep `>= 1` validation and clearly document it, or
   (b) remap `0 → 1` once at the boundary with a comment.

2. `packages/opencode/src/tool/read.ts:248` (post-patch):
   Same `params.offset || 1` swap is passed straight into `lines(filepath, …)`.
   `lines()` is documented as 1-indexed; a literal `0` would have been wrong
   anyway, so the `||` still hides it. Fine in effect, but the change loses the
   "fail loud on bad input" property the deleted guard provided — agents that
   pass negative offsets (`-5`) now get a silent `-5` into `lines()`.

3. The deleted check at lines 154–156 also handled negative numbers
   (`offset < 1`). Post-patch, a negative offset reaches `lines()` /
   `items.slice(start, …)` with `start = offset - 1` becoming a large negative
   index → JS `slice` treats it as offset-from-end, which is surprising for a
   line-reading tool. Recommend re-adding `offset < 0` validation.

4. No test added for the `offset: 0` case the PR title advertises.

## Verdict
**request-changes** — small surface, but the patch as written doesn't actually
support `offset: 0` semantically and removes negative-offset protection without
replacement. Add a test plus a one-line guard for `offset < 0`.
