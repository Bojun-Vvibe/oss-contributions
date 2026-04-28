# QwenLM/qwen-code PR #3684 — feat(core): event monitor tool with throttled stdout streaming (Phase C)

- **PR**: https://github.com/QwenLM/qwen-code/pull/3684
- **Author**: @doudouOUC
- **Head SHA**: `6e4b9300c89642c6b1d5babc094973c3e81e5ba7`
- **Size**: +1663 / −6 across 14 files

## Summary

Phase C of a multi-PR feature stack adding a new built-in `monitor` tool that runs a long-running shell command in the background, intercepts its stdout, and streams throttled event notifications back into the agent context as the command emits matching lines. Adds a new `MonitorRegistry` service mirroring the existing `BackgroundTaskRegistry` / `BackgroundShellRegistry` shape, with `setNotificationCallback` / `setRegisterCallback` hooks for both interactive (`useGeminiStream`) and non-interactive (`nonInteractiveCli`) call paths. Also adds a `sleep`-interception path that *blocks* `sleep N` for `N >= 2` and tells the agent to use the monitor tool instead. Two integration tests in `integration-tests/cli/` exercise both flows.

## Verdict: `request-changes`

Substantial new surface (a new tool, a new registry singleton, a new shell-command interception layer, and a non-trivial change to the non-interactive command-completion logic) shipped without an architecture-level review trail in the PR body. The "Phase C" tag implies this PR depends on Phases A and B which I cannot verify from this PR alone — that's a soft request-changes by itself, but the harder concerns are (a) the `sleep`-interception blocks a *fundamental shell command* with no opt-out shown in the diff, (b) the monitor-registry abort-and-drain protocol in `nonInteractiveCli.ts:736-746` mixes two registries' completion conditions in a way that could deadlock, and (c) the new tool is registered in the global `tool-names.ts` enum without visible permission-gating.

## Specific references

### Sleep interception — blocking fundamental commands without opt-out

- `integration-tests/cli/sleep-interception.test.ts:75-103` — the test asserts that `sleep 5` is **blocked** with a "BLOCKED" message instructing the model to use the Monitor tool. The corresponding production change is in the non-visible-in-this-diff portion of `packages/core/src/tools/shell.ts` (the diff shows +36/-0 there). Blocking `sleep N` for `N >= 2` is a nontrivial UX policy change — `sleep` is the canonical demo command, the canonical CI debug pause, the canonical "don't hammer this API" rate-limit, and the canonical test setup primitive. Replacing it everywhere with "use the monitor tool" assumes the monitor tool is always the right answer, which it isn't (e.g. tests that need a deterministic delay).
- The threshold of `2s` is a magic number with no visible justification. Why not 5s, 30s, only `sleep N` with no other arguments, only when invoked outside of an `&&` chain, only when a parent command isn't expected to block on it? The current shape is a hard cliff that will surprise users.
- **Required**: either a documented opt-out (env var or `--allow-sleep`) and a less aggressive default threshold, or a clear "this only fires inside agent-driven shells, not user-typed commands" carve-out.

### Mixed-registry completion conditions

- `packages/cli/src/nonInteractiveCli.ts:744-749` — the abort-and-drain loop now waits on three conditions:
  ```
  if (
    !registry.hasUnfinalizedTasks() &&
    monitorRegistry.getRunning().length === 0 &&
    localQueue.length === 0
  )
    break;
  ```
  This is correct logic, *but*: `monitorRegistry.getRunning()` returns a snapshot, not a guarantee. If a monitor's stdout-event handler is currently mid-flight pushing onto `localQueue`, all three conditions could be momentarily true at the boundary between "monitor finalized" and "notification enqueued," causing a spurious break followed by a dropped notification. The `BackgroundTaskRegistry`'s `hasUnfinalizedTasks()` was presumably designed to handle this race for itself; the monitor registry needs the same "in-flight notification still pending" signal, not just `getRunning().length === 0`.
- `packages/cli/src/nonInteractiveCli.ts:730` — the abort path calls `monitorRegistry.abortAll()` but the diff doesn't show the implementation of `MonitorRegistry.abortAll()`. If it doesn't synchronously drain the in-flight callback chain or set a state flag readable by `getRunning()`, the wait loop will spin forever after abort.

### Tool registration without visible permission gating

- `packages/core/src/tools/tool-names.ts` (+2/-0) — `monitor` is added to the canonical tool-names enum, making it default-on for every agent. The diff doesn't show a corresponding entry in any allowlist/denylist file, which means the new tool inherits the same default-allowed permission posture as `read_file` and `list_directory`. For a tool that *executes long-running shell commands and streams their output back into model context*, default-on is the wrong posture — it should require the same opt-in gating as `run_shell_command` itself, since it's effectively a wrapper around it.
- The new `monitor.ts` (+389 lines, not visible in the first 250 lines of diff) presumably includes its own permission/confirmation prompts. Need to verify those exist and match the `run_shell_command` confirmation shape before this lands as default-on.

### Other concerns worth addressing

- `packages/cli/src/ui/hooks/useGeminiStream.ts:2255-2269` — registers the monitor notification callback in a `useEffect` that takes `[config]` as its dep array, then unregisters on cleanup. **But**: there's no guard against double-registration if `config` reference-equality changes mid-session, and the cleanup at `:2266-2268` only sets the callback to `undefined` — if the registry was holding a reference to *this hook's* push-into-queue closure, no new closure is wired up before the next event arrives. Race-y.
- `packages/cli/src/nonInteractiveCli.ts:344-368` — registers two callbacks (`setNotificationCallback`, `setRegisterCallback`) on the registry but the cleanup at `:806-810` only unregisters them in the *normal-exit* path. The abort path at `:730-734` calls `abortAll` but doesn't unregister, leaking the closure across sessions if the same `Config` object survives.
- `integration-tests/cli/monitor.test.ts:30-38` — the "should have monitor tool registered" test asks the model to *introspect its own tool list* via natural language ("Reply with just yes or no"). This is a model-as-judge test that will be flaky on weaker models and gives a false-pass on strong models that hallucinate "yes." Replace with a structural assertion against the tool registry (`registry.getTool('monitor') !== undefined`).
- The PR body / commit message would benefit from a one-paragraph "what's in Phase C vs. Phase A/B/D" summary so reviewers can scope what's expected to land here.

## What would unblock

1. Document and bound the `sleep`-interception policy. Either add an opt-out, raise the threshold, or limit to agent-driven shells.
2. Add a "monitor has in-flight notifications" signal to `MonitorRegistry` and use it in the wait-loop instead of `getRunning().length === 0`.
3. Verify `monitor` tool requires the same permission gating as `run_shell_command`.
4. Replace the model-as-judge integration test with a structural registry assertion.
5. Address the callback-leak in the abort path.
