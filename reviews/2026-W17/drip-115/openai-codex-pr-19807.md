# openai/codex#19807 — [codex] Promote Codex App on Windows in tooltips

- PR: https://github.com/openai/codex/pull/19807
- Head SHA: `a37adea0`
- Diff: +45 / -10 across `README.md`, `codex-rs/tui/src/tooltips.rs`, `codex-rs/tui/tooltips.txt`

## What it does
- Replaces the two `IS_MACOS` / `IS_WINDOWS` consts with a single `SUPPORTS_DESKTOP_APP: bool = cfg!(any(target_os = "macos", target_os = "windows"))` in `tooltips.rs:9`.
- Updates `APP_TOOLTIP` and `OTHER_TOOLTIP` strings to mention "Windows and macOS" (`tooltips.rs:11`, `:14`).
- Adds `pick_free_go_tooltip` (`tooltips.rs:112-118`) — a 50/50 RNG split between the existing free-plan promo and the desktop app promo, gated on `supports_desktop_app`.
- Hooks Free/Go users into the new picker at `tooltips.rs:73`.
- Adds two unit tests (`tooltips.rs:383-411` from the diff) — one asserts the rotation produces both tooltips for desktop-supported targets, one asserts the desktop promo never appears on other platforms.
- Tweaks README to say "on macOS or Windows".

## Strengths
- Test coverage matches the new branching exactly: `free_go_tooltip_rotates_between_plan_and_desktop_promos` seeds 16 RNG runs and asserts the resulting set is `{APP_TOOLTIP, FREE_GO_TOOLTIP}`. Good deterministic coverage of a probabilistic branch.
- Consolidating `IS_MACOS || IS_WINDOWS` into `SUPPORTS_DESKTOP_APP` is a clean refactor — every prior `if IS_MACOS || IS_WINDOWS` site now reads as a clear capability check, not a platform check. `paid_app_tooltip` (`:90`) is the most obvious win.
- Falls back to `OTHER_TOOLTIP_NON_MAC` for non-desktop-supported platforms (`:76`) — Linux users don't get a useless "go install the desktop app" prompt.

## Concerns
- The refactor renames the conceptual thing from "IS_MACOS" to "SUPPORTS_DESKTOP_APP" but the constant `OTHER_TOOLTIP_NON_MAC` (`:16`) keeps the old `_NON_MAC` suffix. Now misleading on Windows-supported builds: the name implies "non-mac" but it's used on Linux/etc. Rename to `OTHER_TOOLTIP_NON_DESKTOP` or `OTHER_TOOLTIP_LINUX`.
- `pick_free_go_tooltip`'s 50/50 split is hardcoded with `rng.random_bool(0.5)` (`:114`). If marketing wants to A/B this ratio later there's no knob. Acceptable for now but worth a `// TODO: source from announcement_tip.toml` comment.
- The existing `cargo test -p codex-tui` apparently doesn't go green in the author's env (per PR notes — unrelated stack overflow in `attach_live_thread_for_selection_rejects_unmaterialized_fallback_threads`). The new tests pass in isolation but it would be good to confirm the full suite passes in CI before merge.

## Verdict
**merge-after-nits** — refactor is good, tests are adequate; ask only for the constant rename and a note about CI green-ness.
