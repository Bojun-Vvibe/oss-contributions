# google-gemini/gemini-cli PR #26119 — fix(cli): reset slash-command conflict dedupe when conflicts reappear

- URL: https://github.com/google-gemini/gemini-cli/pull/26119
- Head SHA: ebe22790b549
- Files: `packages/cli/src/services/CommandService.ts`, `packages/cli/src/services/SlashCommandConflictHandler.ts`, `packages/cli/src/services/SlashCommandConflictHandler.test.ts`
- Verdict: **merge-as-is**

## Context

When two slash commands collide (e.g. an extension's `/deploy`
shadows a built-in), the CLI emits a one-time "renamed to
`firebase.deploy`" notification. The dedupe set
(`notifiedConflicts`) prevented duplicate spam during incremental
loading. But the set was never *cleared* when a conflict went
away — so if you uninstalled the conflicting extension and later
reinstalled it (or hot-reloaded the command set), the
notification was permanently suppressed. Issue #24333.

## Design analysis

Two coordinated changes fix this cleanly:

1. `CommandService.ts:51-56` removes the
   `if (conflicts.length > 0)` guard, always emitting a conflict
   event — even with an empty payload — so the handler sees the
   "currently no conflicts" snapshot and can clear stale state.
2. `SlashCommandConflictHandler.ts:48-66` rewrites
   `handleConflicts` to compute `activeKeys = Set<currentKeys>`
   and replace `notifiedConflicts` with `activeKeys` after
   emitting. Conflicts that disappeared between events are
   dropped from dedup; conflicts still present remain deduped.

The `pendingConflicts` filter at line 64 is the subtle correctness
piece: if a conflict was queued for the next debounce flush and
then resolved before the flush fired, it's now removed from the
queue. Without that line, the handler would emit a notification
for a conflict that no longer exists.

The test suite is exemplary. Four new tests pin the spec:

- `should re-notify when a previously-resolved conflict reappears`
  — the headline case from #24333.
- `should not duplicate pending notifications when a conflict
  resolves and reappears before flush` — proves the
  pendingConflicts filter does its job in the resolve→reappear
  race.
- `should re-notify only the conflicts that were resolved and
  reappeared` — A stays deduped while B re-fires; this is the
  test that catches "did you accidentally clear the *whole* set
  on every event".
- The renamed `should debounce newly active conflicts within the
  flush window` — adjusts the debounce semantics test to reflect
  the new "snapshot, not delta" contract.

The new helper `getConflictKey()` extraction at line 73-77 is
trivial but eliminates a duplicated key construction — good
hygiene for a hash-key formula that must not drift between the
two call sites.

## Risks / suggestions

1. The conceptual contract change — events are now *snapshots*
   of currently active conflicts, not *deltas* of new conflicts —
   is now load-bearing on `CommandService` always emitting on
   every refresh. If a future refactor reintroduces the empty-
   payload skip, dedup goes stale again. The new comment at
   `CommandService.ts:52-55` flags this, but a corresponding
   assertion in the test ("CommandService emits even with no
   conflicts") would lock the contract.
2. The `notifiedConflicts = activeKeys` replacement is `O(N)` per
   event. At the slash-command volumes the CLI has, this is
   nothing — but worth a comment if N ever grows.

## What I learned

Dedup state in event-driven code must always have a clear "when
do we forget" rule. "Never forget" works only when the source of
truth is monotonic. Slash-command conflicts are decidedly not
monotonic (extensions install/uninstall, workspaces switch), so
the dedup state must mirror the current snapshot. The shift from
"emit-on-change" to "emit-snapshot-always" plus "dedup =
snapshot" is the same pattern Kubernetes controllers use, and for
the same reason.
