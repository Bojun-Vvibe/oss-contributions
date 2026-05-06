# anomalyco/opencode#25992 — docs: capitalize Neovim in README

- URL: https://github.com/anomalyco/opencode/pull/25992
- PR: #25992
- Author: JGoP-L (JgoP)
- Head SHA: `57c7f08366de02a302ae8894824081a4eaddc9c6`
- State: OPEN  | +1 / -1

## Summary

Single-character README fix: `neovim` → `Neovim` at `README.md:136`. Neovim is the project's [official capitalization](https://neovim.io/) (compare also `Vim` upstream, which the original line gets right by way of "vim users" being lowercased in a different sense). Sister-project naming is `terminal.shop` (intentional all-lowercase brand) and stays untouched, so the diff is appropriately surgical.

## Specific references

- `README.md:136`: `built by neovim users` → `built by Neovim users` (in the bullet listing TUI focus, sandwiched between provider-agnosticism and client/server-architecture bullets).

## Concerns / nits

- None on substance. One-line typo PR.
- Tiny consistency note: the same paragraph still references "Claude Code" with a capital C — fine, since that's also a product name. No collateral capitalization sweeps needed.
- PR has no description body, but the title is self-evident.

## Verdict

**merge-as-is (mas)** — trivial, correct, and matches upstream Neovim's own capitalization convention.
