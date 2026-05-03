# Review: QwenLM/qwen-code PR #3710

- **Title:** feat(cli): customize banner area (logo, title, hide)
- **Author:** chiga0 (ChiGao)
- **Head SHA:** `d2d751e32116d9d7521f5d63e4692693ba7dbc9d`
- **Verdict:** merge-after-nits

## Summary

Adds three `ui.*` settings — `hideBanner`, `customBannerTitle`,
`customAsciiArt` — to let users white-label / personalize the startup
banner without being able to suppress operational data (version, auth,
model, cwd). The PR ships a 1100-line bilingual design doc, a settings
schema addition, the resolution pipeline (`customBanner.ts`), `Header` /
`AppHeader` wiring, and 333 lines of unit tests for the resolver. The
design doc is unusually thoughtful: it explicitly enumerates what is
locked (version, auth/model, cwd) and why.

## Specific-line comments

- `packages/cli/src/ui/utils/customBanner.ts:1-351` — the resolution
  pipeline is well-structured: normalize → read with `O_NOFOLLOW` and
  64 KB cap → strip terminal control sequences → cap at 200 × 200 →
  memoize. The five steps mirror the design doc almost exactly, which
  is a good sign. One concern: `O_NOFOLLOW` is silently dropped on
  Windows (per the design doc); on Windows a malicious symlink in a
  workspace `.qwen/` could still redirect the read. Worth a `[BANNER]`
  warn at startup on Windows when an external `path` is configured,
  so the operator at least sees the trade-off.
- `packages/cli/src/config/settingsSchema.ts:43` (around the new
  `hideBanner` / `customBannerTitle` / `customAsciiArt` block) — the
  three fields follow the existing `hideTips` pattern. `customAsciiArt`
  is `type: 'object'` with `default: undefined`, which is fine, but the
  `description` should mention that the file is read once at startup
  (matches the doc) — users editing `brand.txt` mid-session will be
  surprised otherwise.
- `packages/cli/src/ui/components/Header.tsx` (35/6 diff) — the
  `pickTier(small, large, availableWidth, ...)` algorithm in the
  design doc prefers `large` first then falls back to `small`. Confirm
  the implementation matches: a subtle inversion here would silently
  hide the larger art on wide terminals, which would be very hard to
  diagnose from a bug report.
- `packages/cli/src/ui/components/Header.test.tsx` (46 new lines) — good
  coverage for the locked regions (title version suffix, auth/model
  line, path line stay rendered regardless of customization). Add one
  test asserting that `hideBanner: true` *also* hides the logo column,
  not just the info panel — currently `AppHeader.test.tsx` may only
  cover the wrapper gate.
- `packages/cli/src/ui/utils/customBanner.test.ts` (333 lines) — heavy
  coverage of the resolver paths is the right place to invest. Make
  sure the test that exercises the 64 KB cap uses an in-memory FS or a
  fixture file, not a runtime-generated 64 KB string written to a
  tmpdir, to keep CI fast.

## Risks / nits

- The 1100-line bilingual design doc is excellent context but inflates
  the diff. Consider splitting docs into a follow-up commit so the
  code review is easier to bisect.
- "Hide only the version suffix" is explicitly out of scope, which is
  the right call; expect this to be re-requested by white-label users
  and the PR description should pre-empt that.
- No mention of accessibility: the screen-reader gate already short-
  circuits the banner, but `customBannerTitle` should still flow into
  whatever plain-text alternative the screen-reader path renders.

## Verdict justification

The design is principled (operational data stays locked), the resolver
is defensive (size caps, control-sequence stripping, `O_NOFOLLOW` on
POSIX), and tests cover the resolver thoroughly. The Windows symlink
caveat and the tier-selection direction are worth confirming before
merge but neither is a blocker. **merge-after-nits.**
