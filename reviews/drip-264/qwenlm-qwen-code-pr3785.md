# QwenLM/qwen-code PR #3785 — feat(cli): add memory diagnostics doctor command

- **URL**: https://github.com/anomalyco/qwen-code/pull/3785
- **Head SHA**: `8e886c3b3074`
- **Files touched**: `docs/plans/memory-diagnostics-reference-design.md` (new, ~56 lines), `packages/cli/src/ui/commands/doctorCommand.test.ts` (~+108), `packages/cli/src/ui/commands/doctorCommand.ts`, plus `collectMemoryDiagnostics` in `qwen-code-core`

## Summary

Adds `/doctor memory` (and `/doctor memory --json`) returning a single point-in-time snapshot: `process.memoryUsage()`, V8 heap stats, `process.resourceUsage()`, active handle/request counts, file descriptors when `/proc/self/fd` available, Linux `smaps_rollup`, and a small risk-hint analyzer. Out of scope (per design doc): heap snapshots, polling, retention changes.

## Comments

- `docs/plans/memory-diagnostics-reference-design.md:9-22` — the design doc cites two reference implementations by name. Be careful of trademark/branding text in commit messages and tags going forward; the markdown content here is fine as plain prose comparison.
- `doctorCommand.test.ts:75-78` — `vi.mock('@qwen-code/qwen-code-core', async (importOriginal) => ({ ...(await importOriginal()), collectMemoryDiagnostics: vi.fn() }))` partially mocks a workspace package. If `qwen-code-core` ever lazy-imports `collectMemoryDiagnostics`, the mock won't intercept. A sentinel test that imports the real function and asserts it's a `vi.fn()` would catch a future regression.
- `doctorCommand.test.ts:98-130` — fixture has `analysis.risks: []`. There's a separate test for "successful when risk indicators exist" referenced at line 199 — make sure that test exercises *each* risk hint type at least once (heap pressure, detached contexts, excessive handles, excessive requests, high FD, native memory). Otherwise risk classification can rot silently.
- `doctorCommand.test.ts:147` — `await getMemoryCommand().action!(mockContext, '--json')` parses `--json` as a positional argv string. Confirm that `'memory --json'`, `'--json memory'`, and `'memory --pretty'` (unknown flag) all behave reasonably; the test only covers the canonical happy path.
- The reference design (`memory-diagnostics-reference-design.md:43-52`) lists 4 follow-up PRs. Worth filing tracking issues now and linking from the design doc; otherwise this becomes orphaned scope.
- `collectMemoryDiagnostics` itself isn't in this slice of the diff. Verify (a) it never throws on platforms without `/proc/self/fd` (Windows), (b) the smaps_rollup read uses bounded I/O, (c) repeated calls don't accumulate handles. The "cheap enough to run in normal sessions" claim in the design doc is load-bearing.

## Verdict

`merge-after-nits` — small, well-scoped diagnostic command with a clear non-goals section. Nits are about test coverage of risk-hint paths and flag parsing, not the design.
