# PR #8866 — fix(release): look for .dmg in the right place

- **Repo**: block/goose
- **PR**: #8866
- **Head SHA**: `1ff160016d226d36fcce6f0028a2dc375a568c2f`
- **Author**: jamadeo (Jack Amadeo)
- **Size**: +2 / -2 across 1 file
- **Verdict**: **merge-as-is**

## Summary

Two-line glob fix in the goose2 release workflow:
`release/*.dmg` → `release/**/*.dmg` so the GitHub Release
artifact upload step actually picks up the macOS DMG, which is
produced one directory deeper at `release/dmg/*.dmg` rather than
flat in `release/`. Without this, the release run finishes
"successfully" but the published release has no DMG attached —
classic CI happy-green-with-empty-payload failure mode.

## Specific changes

- `.github/workflows/release-goose2.yml:166` (first artifact
  upload step) and `:184` (second step / draft path) — both
  occurrences updated to `release/**/*.dmg`. The other
  artifact patterns (`release/goose-*.tar.bz2`,
  `release/goose-*.tar.gz`, `release/goose-*.zip`,
  `release/*.exe`, `release/*.msi`, `release/*.deb`) remain
  flat-globbed because they already are produced in `release/`
  directly. So the fix is correctly scoped to *only* the path
  that diverged.

## Risks

1. **Accidental capture of nested artifacts.** `release/**/*.dmg`
   will pick up *any* `.dmg` anywhere under `release/`. If a
   future build step ever produces an intermediate scratch DMG
   (e.g. an unsigned variant or a per-arch staging copy under
   `release/staging/dmg/`), it will be uploaded too. Low
   probability, but worth knowing — a tighter
   `release/dmg/*.dmg` would express the intent precisely. The
   `**/` form is reasonable insurance against future path
   reshuffles though, and the trade-off of "upload one extra
   DMG someday" vs "ship empty release again if path moves" is
   the right call.
2. **Both upload steps use identical patterns by design.** Worth
   confirming the two steps shouldn't ever diverge (one for
   draft, one for full-release? one for nightly, one for
   stable?). If they should always match, a YAML anchor
   (`&artifact_paths` / `*artifact_paths`) would prevent the
   next maintainer from updating one and forgetting the other.
   Out of scope for a 2-line fix, but a useful follow-up.

## Verdict

`merge-as-is` — minimal, correct, scoped to exactly the broken
path. The next release run will validate the fix end-to-end; no
unit-test layer applies for a workflow glob.

## What I learned

CI artifact-upload globs are silent-failure machines: the action
returns success even when zero files match the pattern, and the
only signal is "downstream consumer notices the file is missing
from the release page". The defensive shape — `**/*.ext` for
artifact extensions whose producer-side directory layout is
controlled by an external build tool — costs almost nothing and
catches the entire class of "build moved the file" regressions.
The remaining flat globs in the same step (`*.exe`, `*.msi`,
`*.deb`) are now a latent risk if their build steps ever
restructure; worth a follow-up sweep.
