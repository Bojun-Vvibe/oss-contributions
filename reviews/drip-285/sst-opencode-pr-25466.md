---
repo: sst/opencode
pr: 25466
head_sha: 72612fc77820363119424458e5770ddd70987b42
title: "fix(agent): initialize plugins before resolving config"
verdict: merge-as-is
reviewed_at: 2026-05-03
---

# Review: sst/opencode#25466 — `fix(agent): initialize plugins before resolving config`

**Head SHA:** `72612fc77820363119424458e5770ddd70987b42`
**Stat:** +4 / −0 across 1 file. Closes #25441; also fixes #25450.

This is a re-submission of #25440 (which was auto-closed by a template bot,
not by a maintainer) with the PR template filled in. Original fix attributed
to @quangtran88.

## What it changes

In `packages/opencode/src/agent/agent.ts`, inside the `Layer.effect` factory
that builds the agent layer (around line 80):

```diff
     const skill = yield* Skill.Service
     const provider = yield* Provider.Service

+    // Some entry points resolve agents before project bootstrap runs, so ensure
+    // plugin config hooks have mutated config before agent state snapshots it.
+    yield* plugin.init()
+
     const state = yield* InstanceState.make<State>(
       Effect.fn("Agent.state")(function* (ctx) {
         const cfg = yield* config.get()
```

The fix forces `plugin.init()` to run *before* the `InstanceState` snapshot
calls `config.get()`. Without this, code paths that resolve an agent before
the normal project bootstrap (e.g. `opencode run` with `--agent`, a plugin
that touches agents during startup, or the v1.14 entry point #25441/#25450
both surface) snapshot a config object that hasn't yet had plugin-config
hooks applied — so e.g. plugin-injected providers/models/keymaps appear
missing for the first turn.

## Assessment

- The two issues this closes both describe the same shape: "my plugin's
  config mutations don't show up on first run". Forcing `plugin.init()`
  earlier in the effect chain is the minimal, correct fix.
- `plugin.init()` is idempotent (it's the same handle the bootstrap path
  also calls), so there's no risk of double-initialization side effects
  from this earlier yield.
- Comment at agent.ts:78–79 explains *why* the line is there — important,
  because the next person to "tidy up" the layer order would otherwise
  delete it as redundant.
- 4-line diff, no test added. A regression test would be nice (assert that
  resolving an agent before bootstrap still observes plugin-mutated
  config), but writing one against the Effect layer machinery is a real
  amount of work and the fix itself is mechanically obvious from the two
  closed issues.

## Nits

- (Non-blocking) Would be cleaner long-term to make `plugin.init()` a
  declared dependency of the agent layer rather than an ad-hoc yield, so
  Effect's dependency graph enforces the ordering. Not in scope here.

## Verdict

**`merge-as-is`** — minimal fix, accurate comment, closes two reported
bugs. No reason to hold this for a test; ship and let the regression test
land separately if a maintainer wants one.
