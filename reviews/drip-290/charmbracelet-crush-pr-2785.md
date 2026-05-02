# charmbracelet/crush PR #2785 — fix: limit view size checks to returned content

- Head SHA: `0828d6a4f1d3dcd271b73b7db22f46031af94cf4`
- URL: https://github.com/charmbracelet/crush/pull/2785
- Size: +193 / -10, 3 files (`internal/agent/tools/view.{go,md}`,
  `view_test.go`)
- Verdict: **merge-after-nits**

## What changes

Removes the upfront whole-file size cap in the `view` tool
(view.go:164-168 in the - hunk: `if !isSkillFile && fileInfo.Size() >
MaxViewSize { return ... "File is too large" }`) and replaces it with
a *content-section* size limit applied inside `readTextFile` after
offset/limit are honored (view.go:285-293).

The new contract: `readTextFile(filePath, offset, limit, maxContentSize
int)` accumulates the projected output size as it scans, and returns a
typed `contentTooLargeError{Size, Max}` (view.go:62-68) the moment the
projected emit would exceed `maxContentSize` (view.go:288-292).

The doc string is updated (view.md:3, view.md:21) from "Max file size:
200KB" to "Max returned file content section: 200KB after offset/limit
are applied".

## What looks good

- Fixes a real usability bug: today the tool refuses to read line 5 of
  a 1 MB log file even though the agent explicitly passed
  `offset=4, limit=1`. After this PR, that read succeeds. The
  motivating test `TestViewToolAllowsSmallSectionsOfLargeFiles`
  (view_test.go:148-172) is exactly this scenario and is the right
  regression guard.
- The size accounting at view.go:286-293 correctly accounts for the
  newline separator (`if len(lines) > 0 { projectedSize++ }`),
  matching the eventual `strings.Join(lines, "\n")`. Off-by-one
  avoided.
- `min(limit, DefaultReadLimit)` for slice pre-allocation
  (view.go:281) is a small allocation hygiene win — prevents
  `make([]string, 0, math.MaxInt)` if a caller passes a large
  `limit`.
- Typed error (`contentTooLargeError` + `errors.AsType` at
  view.go:201) is the right call: the caller can format a
  user-facing message that includes the actual sizes
  (view.go:202-204), and any future transport boundary can switch on
  the type instead of string-matching the message.
- The mock infrastructure
  (`mockViewPermissionService`, `mockFileTracker`, `newViewToolForTest`,
  `runViewTool` at view_test.go:225-291) gives future view-tool tests
  a reusable surface — good investment.

## Nits

1. `TestReadTextFileEnforcesMaxContentSize` (view_test.go:178-198)
   passes `MaxLineLength` as the cap. With three lines of
   `MaxLineLength` `a`/`b`/`c` chars the first scan should already
   exceed the cap, which the test asserts (`Empty(t, content)`). Good.
   But the second sub-call passes `offset=2, limit=1` and expects
   "target line" with no error. That works because "target line" is
   <`MaxLineLength`. Add a third sub-call where `offset=0, limit=1`
   and the *first* line is exactly equal to the cap — the boundary
   `projectedSize > maxContentSize` (strict `>`) should allow it,
   which `TestReadTextFileAllowsExactMaxContentSize`
   (view_test.go:206-217) does cover. So the boundary is tested;
   nit: collapse the two boundary tests into a parametric subtest
   for legibility.
2. The error message constructed at view.go:201-203
   ("Content section is too large (%d bytes). Maximum size is %d
   bytes") duplicates the message in `contentTooLargeError.Error()`
   at view.go:67. Either delegate to `err.Error()` (and drop the
   manual format) or have the handler wrap with extra context like
   "after applying offset=%d limit=%d". Right now the duplication is
   a future drift hazard.
3. The skills carve-out switched from a hard size bypass to
   `maxContentSize := MaxViewSize; if isSkillFile { maxContentSize =
   0 }` (view.go:198-201) where `0` means "no cap" (interpreted at
   view.go:289 as `maxContentSize > 0`). That's a fine sentinel but
   worth a comment at view.go:200 — `0` reads like "limit to zero
   bytes" at first glance.
4. `errors.AsType[contentTooLargeError]` at view.go:201 is a generic
   helper presumably from charm's `errors` package; verify it's
   available across the supported Go toolchain versions
   (`charm.land/fantasy` users on older Go). If the project still
   targets Go 1.18+, prefer `errors.As(err, &tooLarge)` for
   compatibility.

## Risk

Low. The change relaxes a constraint (now reads more files) and adds
a new constraint (caps the actual emit). The existing 200KB ceiling
is preserved as the default `maxContentSize`, so no agent that was
working before should regress on memory pressure. The skill-file
unlimited path is preserved.

The behavioral change is exactly the kind of fix that's worth
shipping: agents repeatedly hit the old limit while reading large
log files with surgical offsets, and the old error gave them no
remediation path.
