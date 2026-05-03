# Review: block/goose #8961 — chore(deps): bump rustyline from 15.0.0 to 18.0.0

- **Repo**: block/goose
- **PR**: #8961
- **Head SHA**: `0f7b4c72ae30f276430310470bb093b8c600dd5f`
- **Author**: app/dependabot

## What it does

Dependabot major-version bump of `rustyline` (used by `goose-cli` for
the interactive REPL prompt) across three majors: 15 → 16 → 17 → 18.

## Diff notes

- `crates/goose-cli/Cargo.toml:54` — single line: `rustyline = "15.0.0"`
  → `rustyline = "18.0.0"`. Pinned-exact (vs. caret), consistent with
  how the rest of the crate pins rustyline.
- `Cargo.lock` churn:
  - `rustyline` 15.0.0 → 18.0.0 with `nix` dependency consolidated
    from `nix 0.29.0` → unified `nix 0.31.2` (already in the tree
    via `pty-process`).
  - `fd-lock 4.0.4` removed — rustyline 18 now uses `std::fs::File::lock`
    instead. One fewer transitive dep.
  - `endian-type` 0.1.2 → 0.2.0 and `radix_trie` 0.2.1 → 0.3.0 are
    transitive consequences of rustyline 18.
  - `windows-sys` for rustyline pulls 0.61.2 (was 0.59.0). Other
    crates in the workspace still use 0.59.0, so this introduces a
    second windows-sys version into the build graph.

## Concerns

1. **Three-major-version jump.** The 18.0.0 release notes list a
   non-trivial pile of behavioral changes: minimal repaint (#882),
   "On windows, check that prompt is not styled" (#890 — could
   silently strip ANSI in goose's prompt rendering on Windows),
   `NO_COLOR` env support (#894), `Prompt` trait introduction (#893),
   and signal-handler installation timing (#903). Goose's prompt code
   should be smoke-tested manually on Windows + macOS + Linux before
   merging — Dependabot's compatibility score is not a substitute.
2. The `Prompt` trait at #893 is the most likely API break. Search
   `crates/goose-cli/src/` for `rustyline::Editor`, `Helper`, and any
   custom prompt rendering — if goose implements a `Helper` it may
   need to opt into the new `Prompt` trait or accept default styling.
3. **Two windows-sys major versions** in the build graph (0.59 + 0.61)
   is a bloat regression on Windows builds. Not a correctness issue
   but worth noting; can be cleaned up when the rest of the workspace
   moves to 0.61.
4. No tests touched. Rustyline behavior changes are notoriously hard
   to cover in CI (TTY-dependent), so this is unavoidable; the
   mitigation is a manual three-OS smoke test, not an automated one.

## Verdict

needs-discussion
