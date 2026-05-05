# openai/codex PR #21206 — feat(tui): add ambient terminal pets

- URL: https://github.com/openai/codex/pull/21206
- Head SHA: `df77a410abc8a26ae46c957fd8feedbcde5dabe0`
- Author: fcoury-oai (Felipe Coury)

## Assessment

This is a feature PR adding ambient pets to the TUI. The Cargo.lock churn is the most visible signature: it pulls in `palette_0.7.6` + `palette_derive_0.7.6` for color manipulation, `quantette_0.5.1` + `rand_xoshiro_0.7.0` + `wide_0.8.3` + `safe_arch_0.9.3` for image color quantization, `icy_sixel_0.5.0` for sixel rendering, `bitvec`/`funty`/`radium`/`tap`/`wyz` as bitvec's transitive deps, and `bytemuck_derive` + `by_address` + `fast-srgb8` + `ordered-float_5.3.0` to support the palette/quantization stack. That's a substantial new dependency footprint for "ambient pets," so the size/cold-start cost merits an explicit decision.

The `tui` crate diff (visible at the bottom of `Cargo.lock`: `dependencies = [..., "diffy", "dirs", "dunce", "icy_sixel", "image", ...]` at the codex-tui entry around `:3661-3700`) confirms the new deps land in the TUI binary, not behind a feature flag. For a CLI that emphasizes startup latency and binary size, gating the pet feature behind a `--features pets` cargo feature would let cost-conscious distros opt out. Given that `image`, `palette`, and `quantette` are all release builds (not dev-deps), the binary will grow noticeably; a quick `cargo bloat` comparison before/after would help size the impact for reviewers.

Sixel rendering via `icy_sixel` is terminal-dependent — only a subset of terminals (iTerm2, mlterm, foot, WezTerm, etc.) implement DECSIXEL. The PR diff snippet doesn't show the runtime detection logic, but production rollout needs a graceful fallback for terminals that don't support sixel (most notably the default macOS Terminal.app and most Windows consoles). If the fallback is "render nothing" that's fine; if it's "spew escape sequences as visible garbage" that's a regression. Worth confirming before merge that there's a `term_supports_sixel()` probe gating the render path.

Verdict leans `needs-discussion` because the dependency-footprint and binary-size question is a maintainer policy call, not a code-quality one. The implementation may be perfectly clean — the diff is dominated by lockfile churn so the actual feature code wasn't visible in the truncated output. If maintainers are fine with the deps and the feature is opt-in by default (off unless a config key is set), this can move to merge-after-nits with the sixel-detection check added.

## Verdict

`needs-discussion`
