# block/goose#8998 — feat(analyze): add Elixir support and clarify tool description

- **URL**: https://github.com/block/goose/pull/8998
- **Head SHA**: `5de8cfa93e22`
- **Diffstat**: +71 / -15
- **Verdict**: `merge-after-nits`

## Summary

Adds `tree-sitter-elixir` as a parser for the `analyze` platform extension, registers the `.ex` / `.exs` file extensions, refreshes the tool-description prompt to mention Elixir, and updates the prompt-manager snapshot accordingly. Also bumps every transitive `windows-sys` dependency to `0.61.2` via Cargo.lock churn.

## Findings

- `crates/goose/Cargo.toml` — `tree-sitter-elixir = "0.3.5"` matches the version pulled into `Cargo.lock` (line 11086). Reasonable, actively-maintained crate.
- `crates/goose/src/agents/platform_extensions/analyze/languages.rs` — registers `ex` / `exs` extensions and the parser binding. Worth confirming Elixir's `script` files (`.exs`) and module files (`.ex`) both parse cleanly under `tree-sitter-elixir 0.3.5` against your existing analyze fixtures; if there's no fixture yet, adding even one tiny `defmodule` snippet to the test corpus would lock in the wiring.
- `crates/goose/src/agents/snapshots/goose__agents__prompt_manager__tests__all_platform_extensions.snap` — snapshot updated to include Elixir in the tool description. Make sure this was regenerated via `cargo insta accept` and not hand-edited (whitespace drift in `.snap` files breaks future reviews).
- `Cargo.lock` — the unrelated `windows-sys 0.59.0/0.60.2 → 0.61.2` churn (~10 dependency rows) is incidental. Either call it out in the PR body so reviewers don't grep for the cause, or split it into a separate "chore(deps)" commit. Not blocking, but it bloats the diff and makes bisection harder.

## Recommendation

Feature itself is small and clean. Add a tiny Elixir parse fixture if one doesn't exist, and either justify or split the `windows-sys` churn. Then merge.
