---
pr_number: 2601
repo: charmbracelet/crush
head_sha: 451a99a7f7325ef9978b19ee2049388499ab60db
verdict: merge-after-nits
date: 2026-04-24
---

# charmbracelet/crush#2601 — refresh TUI when session is updated by an external process

**What changed.** Adds a 1-second polling tick to detect cross-process session edits. Three pieces:

1. `internal/ui/model/session.go` (lines 248–290): `scheduleExternalUpdateCheck` (1 s `tea.Tick`) and `checkExternalSessionUpdate` which reads `Workspace.GetSession` and emits `externalSessionChangedMsg` only if `sess.UpdatedAt > knownUpdatedAt`.
2. `internal/ui/model/ui.go` (lines 376, 569–578): wires the tick into `Init`, handles the message in `Update`, only probes when `m.hasSession() && !m.isAgentBusy()` (avoids racing the in-process pubsub broker).
3. New tests `session_test.go` covering: detected change, no-change, timestamp-goes-back, silent-on-error.

**Why it matters.** `crush run -C session-id` from another terminal can mutate session state; the in-process pubsub doesn't see it. The TUI would silently show stale data until the user navigated away and back.

**Concerns.**
1. **1 s polling forever, even when the user is idle and reading.** Per-tick cost is one DB read + one timestamp compare — negligible — but for users with many sessions over a slow filesystem (NFS, network-mounted SQLite) this adds up. Consider backing off when the window is unfocused or the session hasn't changed in N ticks.
2. **`isAgentBusy` gate is correct but fragile.** If the in-process agent crashes mid-turn without clearing busy state, polling stays disabled and external updates stop showing — exact opposite of what you want for "agent died, user wants to recover state." Worth at least a periodic force-probe every 10–30 s regardless of busy state.
3. **`UpdatedAt` is the only freshness signal** (line 285). If two writers within the same second both bump `UpdatedAt` to the same value, the second one is invisible. Sessions are presumably written rarely enough that this is fine, but if `UpdatedAt` is integer seconds, document the resolution.
4. **`loadSession` is called on the change msg** (line 577) — full reload of the message list. For a long session that's expensive. Diffable reloads can come later, but flag it as a known cost.
5. **No test for the busy-state gate.** The whole point of the gate is to avoid double-loading; cover it.
6. **`scheduleExternalUpdateCheck` is fired even when `m.hasSession()` is false** (Init line 376) — first tick will short-circuit fine, but worth noting.

Polling is the right shape for cross-process changes when there's no IPC. Land after the busy-stuck and idle-backoff considerations.
