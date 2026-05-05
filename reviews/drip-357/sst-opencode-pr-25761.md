# sst/opencode #25761 — fix: update file viewer newline handling

- **Repo**: sst/opencode
- **PR**: [#25761](https://github.com/sst/opencode/pull/25761)
- **Author**: kill74
- **Head SHA**: `05575d46c4057f1cf75cba8a600d11b0da3a68d5`
- **Base branch**: `dev`
- **Created**: 2026-05-04T20:47:39Z
- **Closes**: #25704

## Verdict

`merge-after-nits`

## Summary of change

Bumps `@pierre/diffs` from `1.1.0-beta.18` to `1.1.20` to fix a UI bug where
the web file viewer rendered a "No newline at end of file" marker even when
the file *did* end with a newline byte. The marker logic lives entirely
inside `@pierre/diffs`, so the fix is a pure dependency upgrade.

- `package.json:48` — version pin updated `1.1.0-beta.18` → `1.1.20`.
- `bun.lock:695` — workspace dep version updated.
- `bun.lock:1801` — `@pierre/diffs` package entry updated, with new transitive
  `@pierre/theme` pin `0.0.22` → `0.0.28`.
- `bun.lock:1803` — `@pierre/theme` package entry added at `0.0.28`.
- `bun.lock` lines 5653, 6637–6649 — adds a fully-resolved second copy of
  `shiki@3.23.0` (and its `@shikijs/core`, `@shikijs/engine-javascript`,
  `@shikijs/engine-oniguruma`, `@shikijs/langs`, `@shikijs/themes`,
  `@shikijs/types` companions) under the `@pierre/diffs/shiki/` resolution
  prefix. The existing `@shikijs/transformers/@shikijs/types@3.20.0` entry
  is left in place under a different prefix.

## What's good

- Moving off a `-beta.18` release and onto a stable `1.1.20` is the right
  direction regardless of the specific bug — beta-pinned third-party UI
  packages are a long-term maintenance liability.
- The author correctly identified that this is a library-side bug, not a
  consumer-side one, and submitted the upgrade rather than working around it
  in opencode's diff renderer.
- Two-line `package.json` change; the rest is purely the lockfile reflecting
  the resolution.

## Concerns / nits

- **Bundle size impact**: the upgrade pulls in a *second* fully-resolved
  copy of `shiki@3.23.0` alongside the existing `shiki@3.20.0` graph. Per
  the lockfile, both
  `@pierre/diffs/@shikijs/transformers/@shikijs/types@3.20.0` and the new
  `@pierre/diffs/shiki/@shikijs/types@3.23.0` will exist in the workspace.
  Shiki ships a meaningful amount of language/theme data; depending on how
  the web viewer bundles, this could double a substantial chunk. Worth
  measuring `bundle size` before/after and either accepting the regression
  or asking `@pierre/diffs` to widen its `shiki` peer/dependency range to
  let bun dedupe.
- **No verification step shown in the PR description** beyond "the issue
  appears to come from the old beta version". A 30-second confirmation
  attaching a screenshot of the file viewer on a file that ends with a
  newline (showing the marker is gone) and one that doesn't (showing the
  marker is still there) would make this trivially merge-able.
- **No release notes / changelog reference** for the `1.1.0-beta.18 → 1.1.20`
  delta. There's a substantial gap between those two versions; worth a quick
  scan of `@pierre/diffs`'s changelog for any breaking changes to API
  surface that opencode consumes (props names, render output structure,
  CSS class names that opencode might style against).
- **Transitive `@pierre/theme` bump** `0.0.22 → 0.0.28` is included silently
  in the lockfile. If opencode consumes `@pierre/theme` directly anywhere,
  worth verifying compatibility.

## Sanitization

No secrets in the diff. The lockfile contains npm integrity hashes
(`sha512-...`), which are public package metadata, not secrets.

## Risk

Low for the bug fix itself; medium for the bundle-size knock-on. The
behavior change is well-scoped to a UI rendering nit, and the fallback if
`@pierre/diffs` regresses is to pin back to `1.1.0-beta.18`.

## Recommendation

Land after (a) a screenshot or short note verifying the marker behavior in
both the "ends with newline" and "missing newline" cases, and (b) a sentence
in the PR body acknowledging the duplicated `shiki` entry in the lockfile,
or a deduping fix.
