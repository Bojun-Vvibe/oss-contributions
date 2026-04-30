# sst/opencode#25118 — fix(opencode): make sidebar cost display monotonic

- PR: https://github.com/sst/opencode/pull/25118
- Head SHA: `dc39c0aafcb7f2b418e9b69451493b333ab31212`
- Author: adavila0703
- Closes: #25023
- Files: 17 changed, +1585 / −6 (most volume is the auto-generated migration `snapshot.json`)

## Context

Sidebar `$X.XX spent` decreased after compaction or revert because both display sites
(`feature-plugins/sidebar/context.tsx:14` and `routes/session/subagent-footer.tsx:45`)
recomputed the total by reducing over `props.api.state.session.messages(sessionID)` —
the in-memory message cache is capped at 100 messages, so once paid assistant turns
fall out of the window the visible sum drops even though spend hasn't.

## Design

The fix promotes total spend from a derived value to a persisted, monotonically
accumulated session column.

**Schema (`session.sql.ts:40`)** — new `total_cost: real().notNull().default(0)` on
`SessionTable`, accompanied by a real Drizzle migration
`migration/20260430061706_add_session_total_cost/migration.sql` (not just a snapshot).
`real` is the right column type for fractional currency-cents-as-float here; the existing
`cost` field on assistant messages is also a JS number.

**Write path (`processor.ts:377`)** — after `session.updateMessage(ctx.assistantMessage)`
the processor now calls `session.addAccumulatedCost({ sessionID, delta: usage.cost })`.
This is keyed on the same `usage` object that just produced the per-message `cost` field,
so the per-message and accumulated totals can never disagree at the source — they are
both written in the same effect.

**Accumulator (`session.ts:653`)** —
```ts
const addAccumulatedCost = Effect.fn("Session.addAccumulatedCost")(function* (input) {
  if (input.delta === 0 || !Number.isFinite(input.delta)) return
  const current = yield* get(input.sessionID)
  const next = new Decimal(current.totalCost ?? 0).plus(input.delta).toNumber()
  yield* patch(input.sessionID, { totalCost: next, time: { updated: Date.now() } })
})
```
Three things worth calling out: (a) `Decimal.plus(...).toNumber()` avoids the classic
`0.1 + 0.2 = 0.30000000000000004` drift that a naive `+=` would accumulate over thousands
of turns; (b) `Number.isFinite` guards against an upstream NaN/Infinity from a misbehaving
provider torching the running total forever; (c) the `delta === 0` early-return suppresses
spurious update events for free-tier or cached responses.

**Fork path (`session.ts:609-647`)** — `fork()` now walks the source session's messages,
accumulates `forkTotalCost += msg.info.cost` for assistant rows, and patches the new
session's `totalCost` exactly once before returning. This is correct: a fork inherits
historical spend, but the per-message ledger is replicated message-by-message into the
new session, so future deltas land on the new row.

**Read path** — both display sites now read from the persisted column:
- `sidebar/context.tsx:16-19`: `props.api.state.session.get(sessionID)?.totalCost ?? 0`
- `subagent-footer.tsx:45`: `sync.session.get(route.sessionID)?.totalCost ?? 0`

The plugin API gains a corresponding `state.session.get(sessionID)` accessor in
`plugin/api.tsx:170` and the test fixture is updated symmetrically.

**Schema/wire** — `Info` schema (`session.ts:162`) and `UpdatedInfo`
(`session.ts:268`) both gain `totalCost`, the projector
(`projectors.ts:61`) maps `total_cost` ↔ `totalCost`, and the SDK types regenerate.
The schema-decoding test asserts the field round-trips.

## Verdict

**`merge-after-nits`**

The accumulation strategy is sound and the Decimal/`isFinite` guards are exactly what
this code needs to actually be monotonic over a long session. Three nits worth raising
before merge:

1. **Backfill story for existing sessions.** The migration adds the column with default
   `0`. Any pre-existing session opened after upgrade will display `$0.00` until the
   next assistant turn, then jump by `+delta` only. Either run a one-time backfill in
   the migration (sum `cost` from existing `messages` rows per session into
   `total_cost`) or document the discontinuity in the changelog. The current PR does
   neither.

2. **Compaction summary is also an assistant message with non-zero cost.** Worth a
   regression test that compaction increments `total_cost` exactly once with the
   compaction-call usage, not zero (silent drop) and not twice (double-charging the
   summary turn).

3. **Revert path.** The PR fixes the symptom that motivated the bug report, but
   `total_cost` is now intentionally not affected by revert — the spend really happened.
   That's the right call, but it should be a comment on `addAccumulatedCost` or in the
   PR body so a future maintainer doesn't "fix" it by reversing the delta on revert.

The per-message-cost field stays alongside `total_cost`, so analytics that need
per-turn attribution are unaffected.

## What I learned

The pattern here — promoting a derived total from a `messages.reduce()` over a capped
cache to a monotonic persisted column accumulated at the same write site as the
per-row contribution — is the right shape any time the cache size is smaller than the
session's lifetime tail. The two display sites both reduced over the cache; the fix
collapses both to a single source of truth. The `Decimal.plus().toNumber()` detail
matters: in JS, totaling 5,000 fractional-cent deltas with `+=` will visibly drift,
and "the sidebar shows $4.9999999998" looks like a different bug than the one being
fixed.
