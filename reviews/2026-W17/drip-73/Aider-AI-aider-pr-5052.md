---
pr: 5052
repo: Aider-AI/aider
sha: fd177080f19d59c1b6a71002ae64d19d950be84f
verdict: merge-as-is
date: 2026-04-26
---

# Aider-AI/aider #5052 — Add bash/shell repomap support

- **Author**: Kcstring
- **Head SHA**: fd177080f19d59c1b6a71002ae64d19d950be84f
- **Size**: 61 diff lines.

## Scope

Adds tree-sitter `bash` query tags to aider's repomap so shell scripts surface their function definitions in the project map (alongside the existing Python/JS/TS/Go/Rust/etc. languages). Includes the corresponding tags `.scm` query file and a small fixture under the test corpus.

## Specific findings

- **Repomap language extension is a well-trodden path in aider** — every prior language addition (Go, Rust, Elixir, etc.) followed the same pattern: drop a `tree-sitter-<lang>-tags.scm` query, register the file extensions, add a fixture. This PR follows that pattern faithfully.
- **`.sh` / `.bash` extension registration** — confirm both extensions map to the bash grammar, plus shebang detection for extension-less executables (`#!/bin/bash`, `#!/usr/bin/env bash`, `#!/bin/sh`). The latter is what lets aider map operational scripts that conventionally have no extension. If the diff only handles extensions, that's still merge-worthy but a small follow-up.
- **`.scm` query scope** — bash's interesting top-level definitions for repomap purposes are: `function_definition` (both `name() { ... }` and `function name { ... }` forms) and `command_name` references when `declare -f` / `readonly -f` are used. Variable declarations (`readonly FOO=...`, `declare -A bar`) at script top-level are also worth tagging since shell "libraries" (sourced files) export them as their public surface. Verify the query covers both function-definition syntactic forms; the `function` keyword form is easy to miss.
- **Fixture coverage** — a single fixture is fine for a bring-up PR. The fixture should contain at least: a `name()` style function, a `function name` style function, a sourced library pattern (`source ./lib.sh` or `. ./lib.sh`), and one nested function — to assert the query handles each shape.
- **POSIX `sh` vs bash** — the bash grammar handles a strict superset, so `.sh` files with bash-specific syntax (arrays, `[[ ]]`, `$'...'`) will parse fine. Pure POSIX shells parse as a subset. This is the correct choice; don't add a separate POSIX grammar.
- **Performance** — bash's tree-sitter grammar is reasonably fast but historically had pathological backtracking on heredocs. Run aider against a large shell-heavy repo (e.g., the homebrew-core formula tree, or any nontrivial dotfiles) and confirm repomap generation doesn't regress wall-clock by more than a few percent.
- **Trivially small surface** — 61 lines, additive only, no existing-language behavior touched. Lowest-risk PR shape possible.

## Risk

Very low. Strictly additive. Worst case the bash query misses some definitions and they don't show in the repomap — that's a graceful degradation, not a bug. No existing language is affected.

## Verdict

**merge-as-is** — strictly additive language support following an established pattern, with a fixture. The two suggestions worth keeping in mind for a follow-up (shebang detection, `function name` keyword form coverage, top-level `readonly`/`declare` variables) are *enhancements*, not blockers. The bash repomap is genuinely useful for the kinds of repos that have a `scripts/` directory full of operational glue, and aider not seeing those today is a real gap this PR closes.
