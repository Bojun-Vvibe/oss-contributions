# block/goose #8866 — fix(release): look for .dmg in the right place

- **Head SHA**: `1ff160016d226d36fcce6f0028a2dc375a568c2f`
- **State**: OPEN
- **Author**: jamadeo
- **Size**: +2 / -2 across 1 file

## Files changed

- `.github/workflows/release-goose2.yml` (+2/-2)

## What it does

The release workflow's two `softprops/action-gh-release`-style file-glob patterns (one per release artifact upload step) were `release/*.dmg`, which only matches files at the top level of `release/`. The actual upload artifact step earlier in the pipeline puts macOS DMGs at `release/dmg/<arch>/Goose-<version>.dmg`, so the glob was matching nothing and DMGs were silently absent from the published release. Fix swaps both occurrences to `release/**/*.dmg` (recursive glob) so any depth under `release/` is matched.

## Specific observations

- **Two-line fix, two identical changes** at `.github/workflows/release-goose2.yml:166` and `:184`. The two surrounding glob blocks (one for the production release upload step, one likely for a draft/preview release step) are byte-for-byte symmetric otherwise — same `goose-*.tar.bz2`, `goose-*.tar.gz`, `goose-*.zip`, `*.exe`, `*.msi`, `*.deb`, etc. Updating both is correct; updating only one would silently bifurcate the artifact set between draft and published releases.
- **Pattern asymmetry deserves notice**: every other artifact glob in the file uses `release/goose-*.tar.bz2` or `release/*.exe` (top-level only), but the DMG fix moves to `release/**/*.dmg` (recursive). This means the fix admits "DMGs are nested in subdirs because the macOS build step outputs to `release/dmg/...`" while every other build step happens to output its artifact at the top level. If a future Windows-installer build step changes to output at `release/msi/x64/Goose-x64.msi`, the same bug would re-occur on `*.msi`. Worth either (a) standardizing all the artifact globs to `release/**/*.{ext}` for consistency, or (b) standardizing the macOS build step to drop DMGs at the top level. The current state is "globs match the build steps' actual output paths" which is fine but brittle.
- **`release/**/*.dmg` is more permissive than necessary** — it matches `release/dmg/x86_64/Goose-1.32.0.dmg` (intent) and also `release/some-other-tool/random.dmg` if that ever existed (unintended). For a release artifact upload this is unlikely to be a problem (release/ is built fresh per workflow run) but a tighter `release/dmg/**/*.dmg` would express the actual layout. Probably overkill for a 2-line fix.
- **No test coverage possible** for a workflow-yaml glob fix — can only be verified by running the release workflow and observing DMGs in the published artifacts. PR author presumably did this verification (the title implies they hit the bug in a real release). A maintainer doing a dry-run on a fork would be the way to confirm pre-merge.
- **Risk assessment**: lowest-possible-risk change. If the glob now matches *too much*, the release would have extra files; the previous behavior (matched nothing) was strictly worse for the macOS audience. Worst case post-fix: a stray DMG from a debug build leaks into the release, which is a release-process problem upstream of this glob, not a regression caused by this PR.
- **No mention of the inverse**: did the bug previously *block* the macOS release entirely (no DMG = users couldn't install on Mac), or was there a manual fallback (someone was rsyncing the DMG up by hand)? A line in the PR description naming the user-visible impact ("macOS users couldn't install via the release page since vX.Y") would help reviewers calibrate urgency.

## Verdict

**merge-as-is** — minimal, surgical 2-line fix correctly identifies that the macOS DMG output path (`release/dmg/<arch>/`) doesn't match the top-level-only `release/*.dmg` glob, applies the recursive `**` fix to both symmetric upload blocks consistently, and lands with no other behavior change. Optional follow-up to standardize all artifact globs to recursive form to prevent the same bug class re-occurring on future build-step output-path changes, but that's a broader refactor and out of scope for this fix.
