# QwenLM/qwen-code PR #3801 — feat(cli): include monitors in /tasks + add interactive-mode hint

- Author: wenshao
- Head SHA: `e0aea041ff22f1a243a38d654b2348d0b912e63e`
- Diff: +369 / -39 across (per PR body) 2 files: `tasksCommand.test.ts` and `tasksCommand.ts`
- Phase B closure for issue #3634

## Observations

1. **`tasksCommand.test.ts:50-66` adds a `monitorEntry()` factory** that constructs a `MonitorEntry` with realistic defaults (`status: 'running'`, `eventCount`, `idleTimeoutMs: 300_000`, `maxEvents: 1000`, etc.) and a spread for overrides. Mirrors the existing `agentEntry()` helper directly above — good consistency. The `AbortController` field is real (not mocked), which is fine for these unit-level assertions.
2. **`tasksCommand.test.ts:68-87` extends the `beforeEach`** to add `getMonitors = vi.fn().mockReturnValue([])` and wires `getMonitorRegistry: () => ({ getAll: getMonitors })` into the mocked `config` services object. Crucially also sets `executionMode: 'non_interactive'` as the default — this is the test-fixture-level decision that lets the new "interactive hint" assertions in other tests be additive rather than retrofitted. Good.
3. **`tasksCommand.test.ts:182-263` adds the headline test** "lists monitor entries with eventCount and exit / error suffixes" — covers four monitor states (running, completed, completed-with-error/auto-stop, failed) in a single test. The four-way coverage matters because the bug being fixed is "monitors silently disappeared from the listing path", so a test that exercises only the running state would have left the auto-stop and failed branches uncovered. Assertions pin exact strings (`[mon_run] running (12 events)`, `[mon_fail] failed: spawn ENOENT (0 events)`), which catches accidental format drift.
4. **Smart distinction at `test.ts:241-247`** — the `mon_auto` case (auto-stopped at maxEvents) renders `(Max events reached, 1000 events)` *instead of* an exit code. The reviewer comment in the PR diff ("users see WHY the monitor stopped, not just that it did") is the right design principle, and the test pins it. This is the kind of decision that's easy to silently regress when refactoring `statusLabel`, so locking it in via assertion is high-value.
5. **The `// No on-disk output file for monitors`** assertion (PR diff tail) confirms monitors deliberately omit the `output:` line that shells/agents get. This is correct given the architectural difference (monitors stream via `task_notification`, no on-disk artifact) — but it's a behavior contract that's not obvious without context, so the inline comment is well-placed.
6. **Design-decision quality** — PR body explicitly walks through why "keep + interactive hint" was chosen over "delete" or "deprecation hint" from issue #3634, and the reasoning is sound: `non_interactive` (`-p`), `acp`, and SDK consumers genuinely have no Ctrl+T dialog alternative, so deletion would be a silent break. The hint being scoped to `executionMode === 'interactive'` is the right cut.

## Verdict: `merge-after-nits`

Bug fix is real and sharply tested across all four monitor lifecycle states; the surface-redirect design decision is justified explicitly in the PR body and matches the constraints of non-TTY consumers. Pre-merge nit: confirm the actual `tasksCommand.ts` implementation (not visible in the truncated diff above) extracts `statusLabel` / `taskLabel` / `taskId` / `taskOutputPath` cleanly across the three entry types (agent / shell / monitor) — if any of those helpers gained a `kind === 'monitor'` if-ladder rather than a polymorphic dispatch, that's a small refactor worth doing before this lands so future entry types (already hinted at by #3684 / #3791 / #3720) don't compound the conditional.
