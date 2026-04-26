---
pr: 19631
repo: openai/codex
sha: aa9c3d972a2360e0cba66e97f53a5b4175f16189
verdict: merge-after-nits
date: 2026-04-26
---

# openai/codex #19631 — Color TUI statusline from active theme

- **Author**: etraut-openai
- **Head SHA**: aa9c3d972a2360e0cba66e97f53a5b4175f16189
- **Link**: https://github.com/openai/codex/pull/19631
- **Size**: ~708 diff lines across 13 files in `codex-rs/tui/`.

## Scope

Statusline previously rendered as a single `dim()`-styled `Line`. This PR (a) introduces a `(StatusLineItem, text)` segment model preserving item identity, (b) derives per-item foreground accents from the active syntax theme by looking up TextMate scopes through the existing syntax highlighter, and (c) adds a new `AppEvent::SyntaxThemePreviewed` so the theme picker preview/cancel/save paths all refresh the cached statusline colors.

## Specific findings

- `codex-rs/tui/src/app_event.rs:734-737` — `AppEvent::SyntaxThemePreviewed` is added as a no-payload variant. Reasonable, though "Previewed" is past tense for an event that means "preview just changed, please refresh"; `SyntaxThemePreviewChanged` would read more naturally at the dispatch site.
- `codex-rs/tui/src/app/event_dispatch.rs:1714-1729` — `refresh_status_line()` is called in three branches: `SyntaxThemeSelected` Ok path, `SyntaxThemeSelected` Err path (after `restore_runtime_theme_from_config()`), and the new `SyntaxThemePreviewed` arm. Good — this catches the failed-save case which would otherwise leave statusline colors out of sync with the restored theme.
- `codex-rs/tui/src/bottom_pane/footer.rs:640-643` — the wholesale `.dim()` on the passive footer line is removed. The replacement at `:702-712` selectively dims only the separator (` · `) and the active-agent label, keeping the configurable status-line itself at full intensity. This is the right call — blanket dim was the actual visual-flatness complaint — but it changes the on-screen contrast for *every* user, including those who haven't touched their theme. Worth calling out in release notes.
- `codex-rs/tui/src/bottom_pane/chat_composer.rs:3951-3953` — the `.map(Stylize::dim)` is dropped on the composer's combined status line so it picks up the theme-derived colors instead. Consistent with the footer change.
- `codex-rs/tui/src/bottom_pane/footer.rs:1210-1213` and `:1255` — two test-helper render paths drop the `.clone().dim()` chain in favor of just `.clone()`. These are inside `#[cfg(test)]` so they won't change runtime behavior, but they do mean the snapshot tests are now exercising the new (un-dimmed) rendering path. Verify the snapshot files were regenerated, not just the helper.
- The "conservative ANSI fallback when a scope does not provide a foreground" claim from the PR body isn't searchable in the diff slice I inspected — would benefit from a pointer in the description to where the fallback lives so reviewers can confirm low-color-terminal behavior.

## Risk

Medium-low. The functional change is localized to TUI rendering, but the visual change is *global* — every existing user's statusline contrast shifts on upgrade. No data-path or protocol risk. Unit tests at `cargo test -p codex-tui status_line` and `theme_picker` are listed but no behavioral test pins the "preview→cancel restores colors" round-trip that motivated `SyntaxThemePreviewed`.

## Verdict

**merge-after-nits** — add a behavioral test for the preview/cancel restore path (the bug class `SyntaxThemePreviewed` exists to fix), regenerate snapshot tests if not already done, and call out the contrast change in release notes. The architecture (keep status-line at full intensity, dim only chrome) is correct.
