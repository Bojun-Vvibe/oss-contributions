# anomalyco/opencode#25993 — fix(tui): preserve selected model on refresh

- URL: https://github.com/anomalyco/opencode/pull/25993
- PR: #25993
- Author: nexxeln (Shoubhit Dash)
- Head SHA: `ee54e3b3eea3f878c40696db2c73c7091ee48441`
- State: OPEN  | +7 / -15

## Summary

Inverts the auto-set-model-on-agent-change behavior in `packages/opencode/src/cli/cmd/tui/context/local.tsx:397-411`. The pre-patch `createEffect` hook re-derived the active model from `agent.current().model` every time the agent context changed, which meant that on a TUI refresh (where `agent.current()` re-fires) the user's manually-selected model got silently overwritten by the agent's configured default. The patch keeps the validity warning toast for misconfigured agents but **removes the `model.set({ providerID, modelID })` side-effect**.

## Specific references

- `local.tsx:399`: `if (!value) return` → `if (!value?.model) return` (early-out tightened, drops one nested `if`).
- `local.tsx:400-410`: deletes the entire `if (isModelValid(value.model)) model.set({...})` branch. Only the `else` branch (toast warn for invalid model id) survives.

## Concerns / nits

- **Behavioral change worth flagging in PR body**: the previous behavior was *intentional* per the deleted comment `// Automatically update model when agent changes`. New users who configure `model:` on an agent will no longer see it auto-applied when they `/agent <name>` switch — they'll need a separate model command. That's the right call for refresh, but agent-switch is a different code path that gets caught up in the same effect. Worth confirming the maintainer wants both paths changed.
- No regression test pinning "model survives `agent.current()` re-fire". A `createRoot` test that bumps the agent signal twice and asserts `model.peek()` is unchanged would lock the contract.
- Toast `duration: 3000` is hard-coded; project likely has a constant somewhere (low-priority nit).

## Verdict

**merge-after-nits (man)** — correct fix for the refresh case but the agent-switch behavior change should be called out in the PR body and ideally pinned by a test.
