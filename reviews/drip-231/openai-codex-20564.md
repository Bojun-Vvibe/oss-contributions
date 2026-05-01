# openai/codex#20564 — Enforce `animations = false` for screen readers

- **Author:** etraut-openai
- **Head SHA:** `f1016023b900159ddd3d41cf48bb7bd99259f8dd`
- **Base:** `main`
- **Size:** +354 / -45 across 11 files
- **Files changed:** TUI motion module + 8 callsite migrations + status-indicator snapshot + onboarding

## Summary

Closes #20489 — animated TUI affordances (spinners, shimmer text) were leaking through to users who had `tui.animations = false` set, which is the codex setting screen-reader users rely on. Introduces a centralized `tui/src/motion.rs` with explicit reduced-motion fallback choices and migrates every spinner/shimmer call site through it. Adds a regression source-scan test that rejects direct `spinner(...)` or `shimmer_spans(...)` outside the motion module — the canonical "make the right thing easy and the wrong thing detectable" guardrail.

## Specific code references

- `codex-rs/tui/src/motion.rs:18-27`: `MotionMode::from_animations_enabled(bool)` constructs an `Animated` or `Reduced` enum from the existing config flag. This is the single conversion site so every consumer gets the same mapping — the type-system gate that prevents future regressions where a new callsite forgets to plumb the flag through.
- `motion.rs:30-35`: `ReducedMotionIndicator::{Hidden, StaticBullet}` is the load-bearing design choice — different surfaces want different fallbacks. Hook headers want `Hidden` (the spinner prefix is noise without animation; the text alone is meaningful), while exec/MCP/web-search rows want `StaticBullet` (the `•` glyph is the row marker, not just decoration). The `Option<Span<'static>>` return at `:37-49` makes "no indicator" a first-class case rather than a magic empty span.
- `motion.rs:51-62`: `shimmer_text(text, motion_mode)` falls through to `vec![text.to_string().into()]` for `Reduced` (preserving the text as a plain unstyled span) or `Vec::new()` for empty input — correctly avoids emitting an empty span that would inflate the line span count.
- `exec_cell/render.rs:75-82`: the new `activity_marker` adapter wraps `activity_indicator(start_time, MotionMode::from_animations_enabled(...), ReducedMotionIndicator::StaticBullet).unwrap_or_else(|| "•".dim())`. The `unwrap_or_else` path is technically unreachable for `StaticBullet` but provides safe fallback if the enum gains variants — appropriate defensive design.
- `exec_cell/render.rs:107-134`: regression test `active_command_without_animations_is_stable` asserts (a) two successive `command_display_lines` calls return identical output (no time-varying spinner state) and (b) the rendered text is exactly `"• Running echo done"`. The exact-string match is the contract pin — any future regression that re-introduces `◦`-blink or shimmer escape codes will fail this byte-equality assertion.
- `history_cell.rs:1670-1675,1865-1870,2480-2485`: three additional callsite migrations (MCP tool call, web search, MCP inventory loading) all use the same `activity_indicator(..., StaticBullet).unwrap_or_else(|| "•".dim())` shape — consistency is good; arguably the `unwrap_or_else` boilerplate could fold into the `activity_marker` helper.
- `history_cell/hook_cell.rs:631-651`: the hook header path is the most subtle — it uses `ReducedMotionIndicator::Hidden` so the spinner prefix is omitted, then `shimmer_text(hook_text, motion_mode)` returns plain text in reduced mode, then `if !animations_enabled && let Some(span) = header.last_mut() { span.style = span.style.patch(Style::default().bold()) }` re-applies bold to the last span so the reduced-motion text is still visually distinguished. The `let Some` chain (Rust 2024 edition) is clean, and the regression test at `:769-787` pins the exact line `"Running PostToolUse hook: checking output policy"` — no spinner, no shimmer, just bold text.
- `motion.rs:64-78`: `animated_activity_indicator` correctly preserves the original `supports_color::has_16m` branch logic — true-color terminals get the `shimmer_spans("•")[0]` (with safe `.unwrap_or_else(|| "•".into())` fallback for the empty-vec edge case), 8-bit terminals get the time-varying `◦`/`•` blink. No behavior change for animated mode, only the reduced-motion path is new.

## Verdict

**merge-as-is**

This is exemplary accessibility work — closes a real screen-reader regression, centralizes the policy at a single source-of-truth (`motion.rs`), and adds a source-scan regression test (referenced in the PR body) that rejects future direct `spinner(...)` / `shimmer_spans(...)` calls outside the motion module so this regression class can't recur silently. The four reduced-motion regression tests (`active_command_without_animations_is_stable`, `mcp_inventory_loading_without_animations_is_stable`, `visible_hook_without_animations_omits_spinner`, plus the renders-status snapshot update) pin the contract by exact-string equality. The two-axis distinction between `Hidden` and `StaticBullet` reduced-motion fallbacks is the right design — different surfaces have different "what does this row mean without animation" semantics, and exposing that as an enum forces every new caller to make the choice explicitly. Snapshot at `panama_two_lines.snap` was updated as expected. No nits.
