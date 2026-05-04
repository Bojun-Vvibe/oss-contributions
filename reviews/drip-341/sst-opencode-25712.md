# sst/opencode #25712 — feat(tui): show subagent cost rollup in sidebar and task history

- **Head SHA:** `5173697c3ed05b9c5ace33020a50fae6f88d7ab5`
- **Size:** +279 / -5 across 13 files
- **Verdict:** **merge-after-nits**

## Summary
Surfaces subagent (Task tool) LLM spend in the parent session's TUI sidebar and in
the Task block of message history. Adds a server-side `Session.cost(sessionID)` that
BFS-walks descendants and sums assistant message cost across the subtree, exposed via
a new `GET /session/:id/cost` route on both Hono and HttpApi servers. Client side wires
it into the sync store with a `refreshCost` debouncer and a `syncCost(sessionID)`
public API.

## Strengths
- Server route exposed on both Hono and HttpApi keeps backend parity, which the repo
  has been tightening up across recent PRs.
- `inflightCostRefresh` Set in `sync.tsx` (around line 117) is the right shape — it
  prevents the `message.updated` handler from launching N concurrent refreshes when a
  busy subagent emits many assistant updates.
- Refresh trigger is gated on `info.role === "assistant"`, `time?.completed`, and
  `cost > 0`, so cost-free updates do not cause server pressure.

## Nits
- The `refreshCost` swallows all errors silently with `// Ignore transient errors`.
  Consider a low-volume `verbose_logger.debug` (or equivalent) so users with broken
  cost feeds can diagnose without source-diving.
- `for (const id of Object.keys(store.session_cost))` triggers a refresh for every
  cached session on each completed assistant event. For users with long-lived TUIs
  and many parallel sessions, that fans out. A targeted refresh (only ancestors of
  the completed session, or rate-limited window) would be friendlier; acceptable as a
  follow-up.
- Naming: `subagent_count` in the response struct is fine; consider documenting in
  the route that it counts immediate children only vs. the whole subtree (the BFS
  implementation suggests subtree, but the field name reads as direct).

## Recommendation
Land after a brief comment describing the BFS scope and a one-line debug log on
the swallowed error. Targeted-refresh optimization can ship later.
