# Review: sst/opencode #25579 — feat: minimal CLI mode with readline REPL and slash commands

- **Repo**: sst/opencode
- **PR**: #25579
- **Head SHA**: `4acb623a372bf593a1d21c535d7f80d0acce4a07`
- **Author**: iamcheyan

## What it does

Adds an opt-in `--mode repl` that bypasses the TUI and runs a pure
readline loop with 22 slash commands and tab completion, plus a release
workflow to build CLI binaries.

## Diff notes

- `packages/opencode/src/cli/cmd/repl.ts` (+559) — the new REPL entry
  point. Large net-new file; review focus should be on session lifecycle
  parity with the TUI path (does it properly close sessions, handle
  AbortSignal on Ctrl-C, flush pending tool output before prompting
  again?). Without seeing the body, can't verify, but file-size alone is
  a yellow flag for "is this duplicating SDK call sites that already exist
  in `tui/thread.ts`?"
- `packages/opencode/src/cli/cmd/render.ts` (+263) — duplicates tool
  rendering into a non-TUI form. Risk: divergence from the TUI renderer
  over time. Worth asking whether `render.ts` can share a backend with
  the existing tool renderers, or at least extract a shared union type
  for `Inline`.
- `packages/opencode/src/cli/cmd/run.ts` (-211, +4) — meaningful
  shrinkage of `run.ts`; suggests shared logic was extracted into the new
  files. Good.
- `.github/workflows/build-cli.yml` (+40) — release-triggered
  `macos-latest` job that runs `bun run script/build.ts` with
  `OPENCODE_RELEASE=true`. Single OS — the PR description claims
  cross-compile for linux x64/arm64/musl via Bun, which is plausible from
  macOS but the workflow itself doesn't matrix or assert per-arch artifacts.
  Either matrix the OS or document that build.ts handles cross-arch.

## Concerns

1. **Scope**: 559+263 LOC of new surface area as a "feat". This is
   close to a fork of the TUI code path. Maintainers will need to
   commit to keeping REPL parity going forward — that's a non-trivial
   ongoing cost.
2. **No tests** for the new REPL or render module. Slash-command
   parsing and tab completion are exactly the kind of code that
   regresses silently.
3. **Tab completion** (`autocomplete.tsx` +21/-2 in TUI) — touches the
   shared TUI component; that means this PR isn't strictly opt-in. Audit
   that the TUI path is unchanged.
4. The `--mode repl` flag isn't visible in the diff snippet; verify it's
   wired through `bin/opencode` argv parsing.

## Verdict

needs-discussion
