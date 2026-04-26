---
pr: 8826
repo: block/goose
sha: 54de68599e4f593d4ee4afbab5f1bf144e87f166
verdict: needs-discussion
date: 2026-04-26
---

# block/goose #8826 — chore(deps): bump rubato from 0.16.2 to 2.0.0

- **Author**: app/dependabot
- **Head SHA**: 54de68599e4f593d4ee4afbab5f1bf144e87f166
- **Size**: ~125 diff lines (lockfile churn).

## Scope

Dependabot bumps `rubato` from `0.16.2` to `2.0.0`. The PR title says "from 0.16.2 to 2.0.0" but the description body in the metadata also references the upstream changelog from `0.15` → `0.16` → `2.0`. This is a *major-major* version jump (0.x → 2.x), pulling in the new `audio-core` / `audioadapter` crate split that rubato 1.0+ introduced. Lockfile-only churn at the top, but the surface-area change behind it is large.

## Specific findings

- **0.x → 2.x is two breaking-change boundaries.** Per rubato's own CHANGELOG, the 1.0 release reorganized the public API around the `audioadapter` traits and removed `Resampler::process` in favor of `process_into_buffer`/`output_buffer_allocate` patterns. 2.0 then renamed several iterator types and tightened bounds. The fact that the PR is *only* a lockfile bump strongly suggests `goose` is not actually a direct consumer of `rubato` — it's pulled in transitively (probably via `cpal`, `kira`, or one of the speech crates). Verify direct usage with `cargo tree -i rubato` before merge.
- **No source changes implies transitive-only.** If rubato is transitive, the bump only matters when the upstream crate (`cpal`/`kira`/etc.) also bumps to a rubato-2.0-compatible release. If the parent crate still pins `rubato = "^0.16"` or `"^1"`, this lockfile entry won't actually resolve at build time and the PR is a no-op. Check `cargo tree --duplicates` after the bump to confirm there isn't a dual `rubato-0.16.2` + `rubato-2.0.0` graph.
- **Audio resampler quality regressions** are the historical pattern with rubato major bumps — the default `WindowFunction` and sinc-interpolation parameters have shifted across releases. If goose actually plays audio (TTS playback, voice mode), a perceptual A/B test on representative output is required, not just `cargo test`.
- **Wasm compatibility** — rubato 2.0 added optional SIMD via `core_simd`. Confirm goose's wasm builds (if any) still compile; the SIMD feature should default-off there.
- **No CHANGELOG / release note** in the PR body summarizing what 0.16 → 2.0 actually broke. Dependabot's default body is the upstream release links, but a maintainer should add a one-paragraph "what we verified still works" before merge.

## Risk

Medium. Lockfile-only diffs feel safe but mask large API churn. If rubato is genuinely only transitive and the parent crate still resolves to 0.16, this is harmless but also pointless — close it. If the parent has bumped to 2.x and the lockfile is now reflecting that, audio paths need an actual smoke test.

## Verdict

**needs-discussion** — answer two questions before merging: (1) `cargo tree -i rubato` — is goose a direct or transitive consumer? (2) does the bump produce a single resolved version or a duplicated graph? If transitive-only and single-resolved, run a voice-mode / TTS smoke test and merge. If direct consumer, this PR needs source changes too and dependabot's lockfile-only bump is incomplete. If duplicated, hold until the parent crate aligns.
