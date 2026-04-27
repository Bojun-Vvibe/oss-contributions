# QwenLM/qwen-code PR #3671 — banner customization design doc (#3005)

- **PR**: https://github.com/QwenLM/qwen-code/pull/3671
- **Author**: @chiga0
- **Head SHA**: `43c41314d027369d70fd6312460634a167abd8b1`
- **Size**: +1146 / −0
- **Files**: `docs/design/customize-banner-area/customize-banner-area.md` (new)

## Summary

A 1100-line design document — no implementation yet — proposing three new settings (`ui.hideBanner`, `ui.customBannerTitle`, `ui.customAsciiArt`) for the CLI's startup banner. The doctrine is explicit: brand chrome (logo, brand text) is replaceable; operational data (version, auth/model line, working-directory line) is **locked** and cannot be hidden. The doc lays out a region taxonomy, a customization matrix, the three accepted shapes for `customAsciiArt`, the fallback rules, and the rendering contract.

## Verdict: `needs-discussion`

The design philosophy is sound (the lock/replace boundary is the right cut) and the matrix is unusually clean for a CLI customization doc. But two things make this `needs-discussion` rather than `merge-after-nits`: the matrix overcommits to "Locked" cells that real white-label users will push back on, and the doc shipping standalone — without even a sketch of the loader code or the schema update — defers the question of which exact field name lands in `settings.json` to a follow-up that may pick a less considered shape under sprint pressure.

## Specific references

- `docs/design/customize-banner-area/customize-banner-area.md:60-85` — the customization matrix locks the version suffix (`(vX.Y.Z)`), the auth+model status line, and the working-directory line. The rationale is "critical for bug reports / operational signal / footgun prevention". This is the right *default*, but a hard lock is a strong claim — enterprise white-label deployments will absolutely want to suppress the version suffix in screenshots, and the workaround (`--version` only) is not a substitute. A softer doctrine would be: lock by default, expose a single `ui.minimalBanner: boolean` (or admin-only escape hatch) that strips the version-and-status to a single line, and explicitly *not* expose per-field hides. That preserves the "operational data must remain visible somewhere" stance while not foreclosing the white-label use case.
- `:42-58` — the ASCII region taxonomy diagram is genuinely good — it gives names (A, B, B①/B②/B③) to regions that previously had to be referenced by code line numbers. Future PRs about the banner will be much easier to review with this vocabulary.
- `:88-94` — the three settings table is the load-bearing part of the doc. Worth pinning down before merge: (1) is `ui.customAsciiArt` a string (raw text), an array of strings (lines), or a path to a file? The doc says "three accepted shapes" but the diff window doesn't include the resolution rules. (2) what is the expected width budget — does the loader truncate / pad / reject if the custom art exceeds the column count of the info panel? (3) does `ui.hideBanner: true` also hide the `<Tips>` component, or only the banner? The doc says `<Tips>` is governed independently by `ui.hideTips`, but a user who hides the banner probably also expects tips to be quieter — worth a deliberate decision.
- The doc *correctly* keeps the screen-reader fallback (`config.getScreenReader()`) as the existing gate and frames the new `ui.hideBanner` as an additional, parallel gate. That's the right shape — the new setting is additive, not a replacement, so accessibility behavior is preserved.
- The doc references `issue #3005` for the underlying ask, but doesn't summarize what the issue's most-upvoted comment requested. If the issue body asks for "hide the version suffix because we ship to customers under our own brand" and this design says "you can't", that's exactly the gap the design needs to address head-on rather than rationalize.

## Concerns to raise on the PR

1. The "Locked" stance on the version suffix will hit white-label friction. Either soften it to "discouraged but possible via an admin-only escape hatch" or surface, in the doc itself, the explicit answer to "what should an enterprise reskin do?".
2. Pin the three accepted shapes for `ui.customAsciiArt` to concrete schema (string vs. string[] vs. file path) before this design unblocks implementation. The follow-up PR will pick *something*, and the choice should live in the design, not in code review.
3. Add a "rejected alternatives" section: per-field hides, full-replace info-panel template, `--banner-config <path>` runtime flag. A design doc without rejected alternatives is just a spec.

## What I learned

The "lock vs. replace" cut between brand chrome and operational data is exactly the right primitive for any CLI banner customization. Most CLI projects either over-allow (everything is replaceable, debugging tickets become "what version are you on?" three times a week) or under-allow (nothing is customizable, white-label users fork). The matrix in this doc gets the cut right, but locking *too* much and shipping *only* a design without a schema commitment will bite the implementor. This is the kind of PR where the comment thread is more valuable than the merge — the design needs a maintainer pushback round before code starts.
