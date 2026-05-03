# sst/opencode PR #25646 — Effectify plugin agent regression test

- Link: https://github.com/sst/opencode/pull/25646
- SHA: `ee407f1aa88b3dd7107a6d16cf228af177702c67`
- Author: kitlangton
- Stats: +59 / −46, 1 file

## Summary

Rewrites `packages/opencode/test/agent/plugin-agent-regression.test.ts` to run inside the Effect test runtime (`testEffect`) instead of the legacy `WithInstance.provide` callback style. The test now bootstraps `Agent.defaultLayer + InstanceLayer.layer + CrossSpawnSpawner.defaultLayer`, loads an instance via `InstanceStore.Service.use`, and disposes through `Effect.addFinalizer` before the scoped temp directory is cleaned up. Test data was hoisted into a `pluginAgent` const and the two `Bun.write` calls were parallelised.

## Specific references

- `packages/opencode/test/agent/plugin-agent-regression.test.ts` L1–L21: drops `afterEach(disposeAllInstances)` and the `AppRuntime`/`WithInstance` imports; brings in `testEffect`, `InstanceLayer`, `InstanceStore`, `InstanceRef`, and `tmpdirScoped`. Layering all three services through `Layer.mergeAll(...)` is the right move — `Agent.list()` relied on plugin config hooks running first, and the new layer composition keeps that ordering deterministic.
- L23–L27: extracting `pluginAgent` as a typed `const` and reusing it via `JSON.stringify` inside the plugin source means the assertion (`expect(added?.name).toBe(pluginAgent.name)`) and the plugin output cannot drift. Good. Consider also asserting `added?.name === pluginAgent.name` (currently only description/mode are asserted) to guarantee the registered agent really is the plugin's, not a same-keyed default.
- L34–L46: parallelising the two `Bun.write` calls under `Effect.promise(...Promise.all)` is fine because the writes target distinct paths. If a future test adds a third write that depends on plugin.ts existing first, this needs to be re-serialised — worth a one-line comment.
- L48–L57: `InstanceStore.Service.use` plus `Effect.addFinalizer(() => store.dispose(ctx).pipe(Effect.ignore))` is the canonical pattern for ensuring the instance is torn down before the `tmpdirScoped` finalizer removes the directory. The `Effect.ignore` on dispose silently swallows teardown errors — acceptable for a regression test but worth flagging if a flaky dispose ever masks a real bug.
- PR body confirms 3x repeated runs all passed; the previous flake mode (instance not disposed before tmp removal) is what this PR is structurally fixing, so the loop is the right validation.

## Verdict

verdict: merge-as-is

## Reasoning

Pure test refactor with no production-code change. Moves to the canonical Effect pattern, removes the `afterEach`/`AppRuntime` legacy paths, and the dispose-before-tmp-cleanup ordering closes a known flake. The 3x repeat-run validation in the PR body covers the failure mode the change is targeting. Minor nit (assert `added.name`, comment on parallel `Bun.write` ordering) is non-blocking.
