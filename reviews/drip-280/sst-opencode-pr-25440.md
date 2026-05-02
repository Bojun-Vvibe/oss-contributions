# Review: sst/opencode #25440 — fix(agent): initialize plugins before resolving config

- **PR**: https://github.com/anomalyco/opencode/pull/25440
- **Head SHA**: `3b5247cfc373e7c53492e89d43b8e33dece120da`
- **Diff size**: +5 / 0 lines, 1 file (`packages/opencode/src/agent/agent.ts`)

## What it changes

A 3-line surgical fix in `Agent.layer` (`packages/opencode/src/agent/agent.ts:80`). Before
`InstanceState.make<State>` snapshots config, the new code now does:

```ts
yield* plugin.init()
```

Comment claims "some entry points resolve agents before project bootstrap runs, so ensure
plugin config hooks have mutated config before agent state snapshots it."

## Assessment

The fix is correct in shape — `plugin.init()` is what triggers the `config` hook chain that
plugins use to mutate config (env vars, model overrides, agent definitions). If `Agent.layer`
runs before `Plugin.init()` has fired, `config.get()` inside the state factory at line 83
returns un-mutated config and the agent snapshot is wrong. Symptom would be: plugin-defined
agent variants invisible, plugin-injected `permission` rules ignored, etc.

What I want to see before merge:

1. **Idempotency contract**: `plugin.init()` is presumably idempotent (it's already called by
   the bootstrap path). The PR doesn't say so explicitly. If it isn't, calling it from
   `Agent.layer` will double-init plugins that load on bootstrap-then-agent code paths. A
   one-line confirmation in the PR body or a guard inside `plugin.init()` itself would
   close this.
2. **Repro test**: a regression test that resolves an agent via the CLI entry point that
   originally surfaced this bug (looks like a non-`run` entry, possibly `serve`/`tui` cold
   start) and asserts a plugin-injected agent property is visible. Three lines of fix is
   fine; zero coverage on the cold-start ordering bug means the next refactor of
   `AppLayer` will silently re-break this.

The Effect Layer ordering is opaque — the only thing that previously enforced
"plugins-before-agents" was the bootstrap call site. Putting the dependency inside
`Agent.layer` itself (where it belongs) is the right architectural move.

## Verdict

`merge-after-nits` — confirm `plugin.init()` is idempotent in the PR body, and ideally add
a regression test pinning the cold-start ordering.
