# sst/opencode PR #25658 — feat(server): SSE replay buffer with Last-Event-ID support on /global/event

- **PR:** https://github.com/sst/opencode/pull/25658
- **Author:** pasta-paul
- **Head SHA:** `be9beff9` (full: `be9beff97d10d92cbcc1b5653417f652c0a58f2e`)
- **State:** OPEN — closes #25657
- **Files touched:**
  - `packages/opencode/src/server/sse-replay.ts` (+63 / -0) — new ring buffer
  - `packages/opencode/src/server/routes/global.ts` (+50 / -20) — wire Last-Event-ID
  - `packages/app/src/pages/layout/sidebar-items.tsx` (+87 / -32) — sidebar collapsible (out of scope)
  - `packages/app/src/pages/layout/helpers.ts` (+5 / -0)
  - `packages/app/src/pages/layout/helpers.test.ts` (+40 / -0)

## Verdict

**needs-discussion**

## Specific refs

- `packages/opencode/src/server/sse-replay.ts:1-63` — new 1024-entry ring assigning a monotonic id per `GlobalBus` publish. Looks fine in isolation: bounded memory, O(1) append, replay walks from `lastEventId+1`.
- `packages/opencode/src/server/routes/global.ts:50-70` — reads `Last-Event-ID` from request headers, replays buffered entries before transitioning to live. Correctly emits the SSE `id:` field so reconnect cycles can chain.
- `packages/app/src/pages/layout/helpers.test.ts:223-265` — four new unit tests for `childSessions` (sort newest-first, archived exclusion, no-children, exclude-grandchildren). Solid coverage for the helper.
- `packages/app/src/pages/layout/sidebar-items.tsx:97-160` — adds collapsible chevron / `hasChildren` / `expanded` props to `SessionRow`. This is unrelated to the SSE replay fix.

## Rationale

The SSE replay logic itself is the right shape — `Last-Event-ID` is exactly the standard reconnect handshake, and a fixed-size ring keeps memory bounded. Two concerns block "merge-as-is":

1. **Scope creep.** The PR title and body advertise an SSE fix, but ~half the diff is the sidebar collapsible-children UI from the related #25659 branch. These should be split — a server-side reconnect fix and a client-side sidebar redesign have very different review surfaces and rollback profiles.
2. **Buffer eviction window.** 1024 entries sounds large until you consider that a noisy session can emit thousands of `message.part.updated` events per minute during streaming. A client that disconnects for 30s during heavy generation can blow past 1024 and silently fall off the replay window with no signal to the UI. Worth either documenting the limit, surfacing a "replay-truncated" sentinel event, or making the size configurable.

Author should split the sidebar work into a follow-up PR and address the eviction edge case (even if just a doc note) before merge.
