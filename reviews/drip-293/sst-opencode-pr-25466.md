# sst/opencode PR #25466 — fix(agent): initialize plugins before resolving config

- **Repo:** sst/opencode
- **PR:** #25466
- **Head SHA:** `72612fc77820363119424458e5770ddd70987b42`
- **Author:** orbanconsult
- **Title:** fix(agent): initialize plugins before resolving config
- **Diff size:** +4 / -0 across 1 file
- **Drip:** drip-293

## Files changed

- `packages/opencode/src/agent/agent.ts` (+4) — adds `yield* plugin.init()` at line ~80 inside the `Layer.effect` factory, immediately after the `skill`/`provider` service yields and before the `InstanceState.make<State>(...)` call.

## Specific observations

- `agent/agent.ts:80-83` — the comment ("Some entry points resolve agents before project bootstrap runs, so ensure plugin config hooks have mutated config before agent state snapshots it") is exactly what a reviewer needs and exactly what a future grep on this line will need. Keep it.
- The fix is structurally correct: `InstanceState.make` snapshots `cfg = yield* config.get()` inside its constructor (line ~84), so any plugin that mutates config via init hooks must run first or the snapshot is stale. Forcing `plugin.init()` ahead of the state factory closes that race.
- Idempotency concern: `plugin.init()` is called once at agent-layer construction, but `plugin.init()` is also presumably invoked during normal `InstanceBootstrap`. Reviewer should confirm the plugin init contract is idempotent (typical pattern: a `Once` / memoized effect) — if not, this PR will double-init plugins on entry points that *do* run InstanceBootstrap.
- No test added. A regression test would look like: register a plugin whose config hook mutates `agents`, resolve an agent via the affected entry point (e.g. CLI command that doesn't go through full bootstrap), assert the agent reflects the mutation. Worth a small unit test even if just to pin the ordering invariant.
- Diff hygiene is fine — single hunk, no unrelated reordering.

## Verdict: `merge-after-nits`

Correct fix for the snapshot-before-init ordering bug, with a load-bearing comment. Confirm `plugin.init()` is idempotent (or wrap in a `Once` if not), and add a tiny regression test pinning the ordering. Then merge.
