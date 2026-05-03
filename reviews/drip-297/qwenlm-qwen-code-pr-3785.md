# Review: QwenLM/qwen-code PR #3785

- **Title:** feat(cli): add memory diagnostics doctor command
- **Author:** yiliang114 (易良)
- **Head SHA:** `da0f919c091aa1f73303adbefcc8e3c5816fe302`
- **Verdict:** merge-after-nits

## Summary

Adds `/doctor memory` and `/doctor memory --json` subcommands plus a
new `collectMemoryDiagnostics()` core utility. Snapshot covers
`process.memoryUsage()`, V8 heap stats, `process.resourceUsage()`,
active handles/requests, file descriptor count (Linux), and
`smaps_rollup` parsing (Linux). Includes a reference-design markdown
framing this as the first diagnostic layer for tracking issue #3000.
887 added / 5 deleted across 9 files.

## Specific-line comments

- `packages/core/src/utils/memoryDiagnostics.ts:1-272` — the snapshot
  utility is the right shape for first-pass diagnostics: synchronous
  V8 calls + cheap `process.*` reads, and the Linux-only probes
  (`/proc/self/fd`, `smaps_rollup`) live behind `try/catch` with
  graceful undefined returns. The `analysis.risks` array is the right
  separation — collection is mechanical, interpretation is a separate
  layer the user can ignore.
- `packages/cli/src/ui/commands/doctorCommand.ts:1-58` — registers
  `memory` as a real subcommand on `doctorCommand.subCommands`. The
  test at `doctorCommand.test.ts:298-301` (`should register memory as
  a real doctor subcommand`) locks this in.
- `packages/cli/src/acp-integration/session/Session.ts:984-998` — the
  ACP availability flag derivation now honors `cmd.acceptsInput` as
  an explicit override before falling back to the heuristic. This is
  load-bearing for `/doctor` specifically: it has `subCommands` so
  the heuristic would say "accepts input", but `/doctor` itself is a
  no-input command (only its subcommands take args). The
  `acceptsInput: false` override on the parent and the new test at
  `Session.test.ts:280-313` capture this exactly.
- `packages/cli/src/ui/commands/doctorCommand.test.ts:268-280`
  (`should render small memory values without rounding to zero MiB`)
  — good defensive test. `1.0 KB` formatting matters because in unit
  tests the heap is tiny; rounding to `0.00 MiB` would make the test
  output useless. Worth keeping.
- `packages/cli/src/ui/commands/doctorCommand.test.ts:303-358`
  (`should keep memory diagnostics successful when risk indicators
  exist`) — explicit assertion that risk indicators don't fail the
  command. This matches the design note's "risk hints are
  investigation prompts, not failures" stance.

## Risks / nits

- The PR is large (887 LOC) for a "snapshot" feature. Most of that is
  thorough tests + the design doc, which is fine, but reviewers
  should know the actual production code in `memoryDiagnostics.ts` is
  ~272 lines and most of the rest is test scaffolding.
- Heuristic risk thresholds are not documented inline — what counts as
  "excessive handles" or "high file descriptor count" is in code
  somewhere (not visible in the diff excerpt). Consider extracting
  the thresholds to named constants at the top of
  `memoryDiagnostics.ts` so users can sanity-check them and so a
  follow-up PR can make them configurable.
- No coverage of Windows behavior beyond "Linux probes return
  undefined." The Testing Matrix in the PR body marks Windows ⚠️.
  Worth at least one CI smoke that runs `/doctor memory --json` on
  Windows and asserts the schema doesn't blow up.
- `--json` output format is a public surface as soon as anyone scrapes
  it. Add a top-level `schemaVersion: 1` field now so future shape
  changes are detectable.

## Verdict justification

Solid, well-scoped first diagnostic layer with strong test coverage
and an explicit follow-up roadmap. The schema-versioning and
threshold-documentation nits are easy to land before merge but don't
block. **merge-after-nits.**
