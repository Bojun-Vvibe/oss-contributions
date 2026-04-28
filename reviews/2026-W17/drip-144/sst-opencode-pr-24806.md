---
pr_number: 24806
repo: sst/opencode
head_sha: 8e73baed5b5f76c056f78a98413c5031c52b9a07
verdict: merge-as-is
date: 2026-04-28
---

# sst/opencode#24806 — fix: ensure EOL at end of help messages

**What changed.** Single-file, +3/−6 in `packages/opencode/src/index.ts:60-65`. The `show()` helper that writes yargs help to stderr is restructured from an early-return branching pattern (logo-prepend path returned without trailing-EOL guarantee; primary path wrote `out` raw) into a single linear path that (1) computes `output = text.startsWith("opencode ") ? out : UI.logo() + EOL + EOL + text`, (2) writes it, (3) `if (!output.endsWith("\n")) process.stderr.write(EOL)`. Closes #24804.

**Why it matters.** Both paths previously left it to yargs to terminate the help block with a newline. yargs trims trailing whitespace on subcommand help output (the `bun dev stats --help` regression in #24804), which left the shell prompt glued to the last `--agent`/`--project` line. The minimal correct fix is to make EOL invariant a property of `show()` itself rather than its caller — exactly what this diff does.

**Concerns.**
1. **The `endsWith("\n")` check is correct, not `endsWith(EOL)`.** On Windows `EOL === "\r\n"`, but yargs always emits `\n` for line terminators in its help text regardless of platform, so `endsWith("\n")` is the right invariant — `endsWith(EOL)` would falsely fail on yargs-produced Windows output and double-write `\r\n\r\n`.
2. **`text.startsWith("opencode ")` is the discriminator** distinguishing the `opencode --help` top-level case (already starts with the brand line, no logo prepend) from subcommand help (`opencode stats --help` → text starts with `stats <project>`). Same predicate as before, only the control flow collapses.
3. **Diff is byte-symmetric with the bug class** — only the missing-trailing-EOL invariant is added; logo-prepend semantics, stream choice (stderr), and `trimStart()` behavior are byte-identical. No collateral risk.
4. **No test added.** Trivial enough that the screenshot diff in the PR body is sufficient evidence, but a one-liner test capturing `process.stderr.write` calls and asserting last-call-ends-with-newline would prevent regression on the next refactor of `show()`.
5. **The PR description's own checklist box "I have not included unrelated changes"** is honest — single file, 9 lines net, focused on one symptom.

Ship.
