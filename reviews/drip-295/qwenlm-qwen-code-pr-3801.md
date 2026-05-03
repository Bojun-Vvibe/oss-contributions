# QwenLM/qwen-code PR #3801 — feat(cli): include monitors in /tasks + add interactive-mode hint

- **Repo:** QwenLM/qwen-code
- **PR:** #3801
- **Head SHA:** `796bd3aead8b5e6dc789a8b2fe19eac25be3d34d`
- **Author:** wenshao
- **Title:** feat(cli): include monitors in /tasks + add interactive-mode hint
- **Diff size:** +217 / -13 across 2 files
- **Drip:** drip-295

## Files changed

- `packages/cli/src/ui/commands/tasksCommand.ts` (+77/-13) — adds a third `MonitorTaskEntry` arm to the `TaskEntry` union, renders monitor entries with `(N events)` suffix and `pid=` line, gates a new `Ctrl+T` hint on `executionMode === 'interactive'`, and replaces the binary `kind === 'agent' ? ... : ...` ternaries in `taskId`, `taskOutputPath`, and `taskLabel` with explicit three-arm dispatch.
- `packages/cli/src/ui/commands/tasksCommand.test.ts` (+140/-0) — four new tests: monitor rendering with `pid`/`exitCode`/`error`/auto-stop, singular/plural pluralization (`1 event` vs `1 events`), interactive-mode-only hint, and merged-sort ordering across all three task kinds.

## Specific observations

- `tasksCommand.ts:254-270` — the new `taskId` switch with the `_exhaustive: never` guard is the right shape for a discriminated union; the `default` arm panics with the offending entry stringified, which will surface a useful error at runtime if a fourth `kind` is added without updating this switch. Good.
- `tasksCommand.ts:272-279` — `taskOutputPath` returns `undefined` for monitors with the comment "Monitors stream to the agent via task_notification rather than a file on disk". That's load-bearing — the rendering loop conditionally appends an `output:` line only when this is defined, and the test at line 115 (`expect(result.content).not.toContain('output: ')`) pins it. Solid.
- `tasksCommand.ts:222-240` — the monitor `statusLabel` switch handles `completed` two ways: with `entry.error` (auto-stop, eg. "Max events reached") it formats `(error, N events)`; without it formats `(exit ?, N events)`. But monitors don't have `exitCode` in the type signature shown — they have `error` set on auto-stop completion. The `exit ${entry.exitCode ?? '?'}` branch will therefore always render `exit ?` for completed monitors that lack an error. Either drop the `exit ...` branch entirely for monitors or fall back to a clearer "completed (N events)" string when neither `error` nor a real exit code is available. The test at line 105 (`'[mon_done] completed (exit 0, 7 events)'`) only passes because the test fixture explicitly sets `exitCode: 0` on the monitor entry — confirm `MonitorEntry` actually carries `exitCode` in production; if not, the test fixture is misleading.
- `tasksCommand.ts:221` — `const events = \`${entry.eventCount} event${entry.eventCount === 1 ? '' : 's'}\`` — pluralization is correct and the test at line 126 explicitly guards against `"1 event"` matching the prefix of `"1 events"`. Minor: `0` events renders as `"0 events"` (correct English). No issue.
- `tasksCommand.ts:332-336` — `pidPart` now includes `monitor` in addition to `shell`. The `MonitorEntry` type from `@qwen-code/qwen-code-core` must therefore expose an optional `pid?: number`. The test fixture (line 17-32 of the test file) doesn't set `pid` by default but the assertion at line 102 requires `pid=9999` for the explicitly-pid'd entry. Confirm the type lives where the import expects.
- `tasksCommand.ts:294` — `supportedModes: ['interactive', 'non_interactive', 'acp']` — keeping the slash command available across all three modes is correct because headless `-p` consumers have no dialog to fall back on. The comment explains this clearly. Worth lifting into a doc page.
- `tasksCommand.ts:316-325` — the Ctrl+T hint is gated on `executionMode === 'interactive'`. Good. But the hint string ("Ctrl+T opens the interactive Background tasks dialog...") will render on Windows where the actual binding may differ from Ctrl+T (terminal capture, conflicts with browser shortcuts). If the binding is configurable, source it from the keybind registry instead of hardcoding "Ctrl+T".
- The mock context at test-file `:44-53` introduces `executionMode: 'non_interactive'` as a default. If `createMockCommandContext` previously defaulted to `'interactive'`, every existing test in this file is now running under a different mode by accident. Confirm the prior default — if it was interactive, prior tests should be re-checked to ensure they don't inadvertently rely on interactive-only behavior.
- `tasksCommand.test.ts:154-175` — the merged-sort test is the most useful one in this PR; pinning the cross-kind ordering by `startTime` prevents a silent regression if a future change groups by kind first. Good.
- The diff has thoughtful inline comments justifying each design decision (the `// monitors stream via task_notification` block, the `// Soft redirect` block). That's atypical and pleasant; preserve it through review.

## Verdict: `merge-after-nits`

Two real things to clean up before merge: (1) the `completed (exit ${entry.exitCode ?? '?'}, ${events})` branch for monitors will print `exit ?` in the common case unless `MonitorEntry` actually carries `exitCode` — verify the type and either drop the branch or document; (2) the hardcoded "Ctrl+T" string should source from the keybind registry if one exists. The exhaustive-switch refactor and test coverage are solid.
