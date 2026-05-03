# QwenLM/qwen-code #3801 — feat(cli): include monitors in /tasks + add interactive-mode hint

- **PR:** https://github.com/QwenLM/qwen-code/pull/3801
- **Head SHA:** `e0aea041ff22f1a243a38d654b2348d0b912e63e`
- **Author:** wenshao
- **Size:** +369 / -39 across 2 files
- **Phase B closure for:** Issue #3634

## Summary

Two coupled changes to the `/tasks` slash command:

1. **Bug fix** — `/tasks` was last touched before #3684 / #3791 landed monitor entries, so it merged only agent + shell entries. Monitors silently disappeared from headless / non-interactive / ACP listing paths. Adds `getMonitorRegistry()` plumbing.
2. **UX** — adds an interactive-mode hint when the user runs `/tasks` outside an interactive TTY, surfacing how to attach to a monitor.

## Specific references

- `packages/cli/src/ui/commands/tasksCommand.test.ts:11-12` — imports `MonitorEntry` type. Required because the new test factory builds full `MonitorEntry` shapes.
- `tasksCommand.test.ts:48-65` — new `monitorEntry()` test factory with sensible defaults: `monitorId: 'mon-aaaaaaaa'`, `command: 'tail -f app.log'`, `status: 'running'`, `eventCount: 0`, `maxEvents: 1000`, `idleTimeoutMs: 300_000`, `droppedLines: 0`. Matches the runtime monitor shape introduced in #3684.
- `tasksCommand.test.ts:75-85` — `getMonitors = vi.fn().mockReturnValue([])` and the new context wires `getMonitorRegistry: () => ({ getAll: getMonitors })`. The empty-array default is the right sentinel for the "no monitors" path.
- `tasksCommand.test.ts:179-380` (new `it('lists monitor entries with eventCount and exit / error suffixes')`) — exercises three monitor states: `running` (with pid + eventCount), `done` (exit code), and an error case. Good coverage of the rendering logic.
- `tasksCommand.ts` — implementation pulls monitors via `getMonitorRegistry()` alongside the existing agent + shell registries.

## Concerns

1. **`executionMode: 'non_interactive'`** is hard-coded in the test setup. Worth a parallel test asserting the interactive-mode hint is *not* shown when `executionMode` is interactive (the PR's stated UX behavior). If only the non-interactive path is covered, the hint visibility under interactive mode is untested.
2. The `monitorEntry` factory uses `Date.now() - 4_000` for `startTime` — fine for a snapshot test, but fragile if any rendering logic includes "started X seconds ago" formatting (would need clock mocking to stay deterministic).
3. The test factory invents `pid: 9999` for the running case — confirm the actual `MonitorEntry` type marks `pid` optional (the factory's spread suggests yes). Otherwise the cast would warn.

## Verdict

**merge-after-nits** — correct fix for a real regression introduced when monitors were added without updating the slash command. Nits: add the interactive-mode counterpart test and consider mocking the clock if "ago" rendering is involved.
