# openai/codex PR #19807 — promote desktop app on Windows in tooltips

- **PR**: https://github.com/openai/codex/pull/19807
- **Author**: @vb-openai
- **Head SHA**: `a37adea0db55681b873443b340fb68c4f88b3b77`
- **Size**: +45 / −10
- **Files**: `README.md`, `codex-rs/tui/src/tooltips.rs`, `codex-rs/tui/tooltips.txt`

## Summary

Collapses the previous `IS_MACOS` / `IS_WINDOWS` cfg gates into a single `SUPPORTS_DESKTOP_APP` constant, threads it through the four tooltip-selection sites that previously asked only about macOS, and adds a 50/50 rotation between the plan-promo and app-promo on Free/Go plans for desktop-supported targets. Two table-test stubs lock both branches in.

## Verdict: `merge-as-is`

A tight, mechanical refactor that fixes a real correctness bug (Free/Go-plan Windows users were seeing only the generic plan-promo and never the desktop-app-promo because the `paid_app_tooltip` path was the only one consulting `IS_WINDOWS`). Behavior on non-supported platforms is preserved by the explicit `OTHER_TOOLTIP_NON_MAC` arm and the new `pick_free_go_tooltip(rng, /*supports_desktop_app=*/ false)` test.

## Specific references

- `codex-rs/tui/src/tooltips.rs:8-9` — `const SUPPORTS_DESKTOP_APP: bool = cfg!(any(target_os = "macos", target_os = "windows"));` is the right primitive. Single source of truth, evaluated at compile time. The previous `IS_MACOS` / `IS_WINDOWS` constants are deleted entirely (good — no temptation to drift them apart later).
- `codex-rs/tui/src/tooltips.rs:30` — the `tooltips.txt` filter that suppressed `"codex app"` lines on non-supported platforms is correctly migrated from `!IS_MACOS && !IS_WINDOWS` to `!SUPPORTS_DESKTOP_APP`. Identical truth table; clearer intent.
- `codex-rs/tui/src/tooltips.rs:73` — Free/Go arm now calls `pick_free_go_tooltip(&mut rng, SUPPORTS_DESKTOP_APP)` instead of the unconditional `FREE_GO_TOOLTIP`. This is the user-visible behavior change: 50% of Free/Go-plan banner impressions on macOS *and* Windows now rotate to the desktop-app promo. On Linux/other, the helper is hard-wired to return `FREE_GO_TOOLTIP`.
- `codex-rs/tui/src/tooltips.rs:380-410` — two `#[test]` functions: `free_go_tooltip_rotates_between_plan_and_desktop_promos` (asserts both strings are seen across 16 seeds when `supports_desktop_app=true`) and `free_go_tooltip_skips_desktop_promo_on_other_platforms` (asserts only `FREE_GO_TOOLTIP` is seen when `supports_desktop_app=false`). The 16-seed sweep is enough that a bias-toward-one-branch regression would fail with high probability.

## Nits

None worth blocking on. One micro-observation: `OTHER_TOOLTIP_NON_MAC` is now misnamed (it's "non-desktop-app", not "non-Mac" since Windows now goes through the `OTHER_TOOLTIP` arm). A follow-up rename would close the naming-vs-behavior drift, but it's not worth churning this PR.

## What I learned

The bug shape — "we have two cfg-flagged platforms but only ever check one" — is the kind of mistake that's invisible at code-review time and only surfaces when product asks "why isn't Windows seeing the new banner?". The fix is to give the union a name (`SUPPORTS_DESKTOP_APP`) and ban the individual `IS_OS` constants from the file. Worth stealing as a lint-rule heuristic: if you have two `cfg!(target_os = ...)` constants whose disjunction is what you actually mean, name the disjunction.
