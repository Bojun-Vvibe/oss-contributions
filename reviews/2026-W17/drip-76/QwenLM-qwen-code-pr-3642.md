---
pr: 3642
repo: QwenLM/qwen-code
sha: c8d7d151a1b6a6e8bbb58e08e20fffe7b7ea29ca
verdict: merge-after-nits
date: 2026-04-26
---

# QwenLM/qwen-code #3642 — feat(core): managed background shell pool with /bashes command

- **URL**: https://github.com/QwenLM/qwen-code/pull/3642
- **Author**: wenshao (Shaojin Wen)
- **Head SHA**: c8d7d151a1b6a6e8bbb58e08e20fffe7b7ea29ca
- **Size**: +791/-406 across 9 files
- **Related**: #3076 (background subagents), #3471 (BackgroundTaskRegistry), #3488 (footer pill), #3634 (background tasks roadmap — Phase B)

## Scope

Replaces the legacy `&` fork-and-detach background path in `shell.ts` with a managed `BackgroundShellRegistry` that gives observable lifecycle to background processes. Three product surfaces:

1. **`BackgroundShellRegistry`** in `packages/core/src/services/backgroundShellRegistry.ts`: per-entry status state machine (`running` → `completed` | `failed` | `cancelled`), one-shot terminal transitions, AbortController, output file path. Mirror-shape of #3471's `BackgroundTaskRegistry` for future unification.
2. **`shell.ts` `executeBackground` rewrite**: spawns the unwrapped command (no `&`, no pgrep envelope, no Windows ampersand-cleanup), streams stdout to `<projectDir>/tasks/<sessionId>/shell-<id>.output`, bridges external abort signal into the entry's `AbortController`, returns immediately with `{id, outputPath}`. Settles the registry entry asynchronously.
3. **New `/bashes` slash command** at `packages/cli/src/ui/commands/bashesCommand.ts` (registered in `BuiltinCommandLoader.ts:95`): lists registered shells with id, status, runtime, command, output path. Empty state.

Removes ~120 lines of dead bg-specific code from `shell.ts`.

## Specific findings

- **Slash-command registration is alphabetically misplaced.** At `BuiltinCommandLoader.ts:95`, `bashesCommand` is inserted between `agentsCommand` and `arenaCommand`. Lexicographically `bashes` sorts after `arena` (`a-g-e < a-r-e < b-a-s`), so the correct slot is after `authCommand` or wherever `b*` commands would land. Looking at the existing list (alphabetical-ish), the right insertion point is near other `b`-prefixed commands. Minor — the loader iterates the list as-is; this only affects help-text ordering — but worth fixing in the same diff.

- **Test `entry()` factory at `bashesCommand.test.ts:14-25` correctly constructs both running and terminal entries.** The state-machine fields (`pid?`, `exitCode?`, `endTime?`, `error?`) are all properly optional. The "lists running and terminal entries" test at `:50-91` exercises `running`, `completed (exit 0)`, and `failed: <message>` rendering paths. Solid coverage.

- **Output-path layout `<projectDir>/tasks/<sessionId>/shell-<id>.output`** aligns with the direction sketched in #3471 review (per PR description). Using `tasks/` as the parent directory (rather than `bashes/` or `shells/`) is the right call — it sets up the future unification of #3471's task registry and this PR's shell registry under one filesystem hierarchy.

- **One-shot terminal transitions** are the right design. The PR description claims "late callbacks no-op" — this prevents a slow `complete` callback from racing a user-triggered `cancel` and ending up with the registry in `completed` state when the kill signal already fired. Worth verifying the unit tests cover the race: a transition from `running` → `cancelled` followed by a late `complete` callback should leave the entry as `cancelled`, not flip to `completed`. (PR description says "11 registry unit tests (state machine + idempotent terminal transitions)" so this is likely covered; reviewer should spot-check.)

- **Abort-signal bridging** ("Bridges the external abort signal into the entry's `AbortController` so a single source of truth governs cancellation") is the architecturally correct choice. The previous `&`-fork-and-detach path had no kill path at all — the PID was lost the moment the fork returned. Now `entry.abortController.signal` is the only thing that matters, and external abort listeners get bridged in via `signal.addEventListener('abort', () => entry.abortController.abort())`. Standard pattern; right answer here.

- **`/bashes` empty state at `bashesCommand.ts` returns `'No background shells.'`** — short and informational. Good. But there's no way for the user to discover the relationship between `/bashes` and the existing `tool: shell` background mode without reading the docs. Worth adding a one-line hint to the empty-state message: "No background shells. Use `is_background: true` on shell tool calls to start one." Optional.

- **Removal of pgrep wrapping + Windows ampersand cleanup + Windows early-return path** is the cleanup-of-dead-code half of this PR. Reviewer should spot-check Windows behavior — the previous Windows early-return existed because `&` background-spawn semantics differ on Windows. With the new `executeBackground` path, Windows now goes through the same `child_process.spawn` codepath as Unix, which should work but is the most likely regression surface. Manual smoke on Windows would be reassuring.

- **9 files changed for what's described as a focused refactor** — the file list isn't fully visible but the diff scope (+791/-406, single-feature) is appropriate. Test files account for ~290 lines (registry + bashesCommand + shell.test.ts background-path tests) which is a healthy ratio.

- **Author signal is strong:** wenshao has prior PRs in this repo (e.g. #3637 from drip-74), this is Phase B of the documented #3634 roadmap, dependencies on #3471/#3076 are explicitly called out, and gating on #3488 / #3471 for footer-pill / `task_stop` integration is clearly deferred. Disciplined sequencing.

## Risk

Medium. The registry shape, output-path layout, and `/bashes` UX are all forward-compatible with the planned #3471 unification, so this isn't a one-shot design that'll need to be ripped out. The main risk is Windows: the legacy code had Windows-specific paths that are being deleted, and the new spawn path needs to handle Windows pipe semantics (`stdout.pipe(createWriteStream(outputPath))` may need `{ end: false }` on Windows to avoid premature stream closure). PR description says "Full core suite: 247 files / 6075 passed" — that's CI on (presumably) Linux. Windows manual smoke is the gap.

## Nits

1. Re-slot `bashesCommand` insertion at `BuiltinCommandLoader.ts:95` to a `b`-prefixed alphabetical position.
2. Optional: extend `/bashes` empty-state message with a discovery hint about `is_background: true` shell tool calls.
3. Confirm Windows manual smoke on `executeBackground` (long-running command + abort + output-file flush).
4. Spot-verify that the registry's "late callbacks no-op" guard covers the `cancel-then-late-complete` race (PR description implies a test exists; flagging for reviewer attention).

## Verdict

**merge-after-nits** — well-staged refactor that lands real user-visible improvements (observable bg shells, `/bashes` listing, abort-signal coherence) while paving the way for the documented #3471/#3488 follow-ups. The four nits are polish, not correctness blockers.

## What I learned

The right way to unwind a "fork-and-detach" anti-pattern is to introduce a registry first (one-shot state machine, AbortController-as-single-source-of-truth, observable lifecycle), then rewrite the spawn-site to register itself rather than detach. Doing it in the other order — rewriting the spawn first and bolting on observability later — leaves a window where existing call-sites depend on the detach semantics. Wenshao's sequencing (registry first in `services/`, then `shell.ts` rewrite that consumes it) is the textbook ordering. The Phase A/B/C/D roadmap from #3634 making each step independently shippable is also worth copying — most refactors of this size collapse the phases together and become un-reviewable.
