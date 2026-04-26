---
pr: 24452
repo: anomalyco/opencode
sha: 5c5a85fb2f137d58c446adaa697663ef82b80184
verdict: merge-after-nits
date: 2026-04-26
---

# anomalyco/opencode #24452 — feat(tui): pinned session tabs in the right sidebar

- **Author**: MrRobotoGit
- **Head SHA**: 5c5a85fb2f137d58c446adaa697663ef82b80184
- **Link**: https://github.com/sst/opencode/pull/24452
- **Size**: +398 / -17 across 5 files. New context `context/session-tabs.tsx` (214 LOC), new sidebar route component `routes/session/session-tabs.tsx` (104 LOC), `app.tsx` provider wiring + 4 new commands, `keybinds.ts` (+4 binds).

## Scope

Adds a pinned-tab UI to the session route. State model: `pinned[]` (persisted to `Global.Path.state/session-tabs.json`, keyed by directory) plus `visited[]` (in-memory, cleared each TUI run). Visible list = `pinned ∪ visited(this-run, exists) ∪ busy/retry`, deduped, pinned first. Four new commands: pin/unpin active, close active tab, next/prev tab.

## Specific findings

- `context/session-tabs.tsx:38-46` — store has `ready` flag; reads `session-tabs.json` on init and only clears `pending` after `setStore("ready", true)`. The `persist()` early-out at `:46-50` correctly defers writes during the read, but: if `persist()` is called twice before `ready`, the second call silently no-ops the first's pending intent (it just re-sets `pending=true`). For this surface that's fine because `persist()` always rewrites the full pinned list, but worth a comment.
- `context/session-tabs.tsx:60-68` — read path filters non-string ids with `pinned.filter((id) => typeof id === "string")` but does NOT validate that the id corresponds to an existing session. Stale ids from deleted sessions (e.g. external deletion between runs) will appear in `pinned` until the user touches them. The `event.on("session.deleted", ...)` at `:75` only handles deletions during this run. Consider a startup pass that drops `pinned` ids not present in `sync.session`.
- `context/session-tabs.tsx:76-84` — `createEffect` adds the active session id to `visited` whenever the route is `session` and id is truthy + not "dummy". This effect re-runs on every store change in its dependency graph; SolidJS will dedupe, but the explicit guard `if (store.visited.includes(id)) return` is O(n) per nav. Fine at small N (typical pin counts <20), no action needed.
- `context/session-tabs.tsx:104-119` (`removeAndAdjust`) — when the removed pin was active, fallback chain is `next[idx] ?? next[idx - 1] ?? next[0]`. Edge case: if the removed pin was the only tab AND there are no ephemeral/busy sessions, `fallback` is `undefined` → `navigateTo(undefined)` → `route.navigate({ type: "home" })`. That's the desired behaviour. Good.
- `context/session-tabs.tsx:140-162` — `visible` memo recomputes on `pinned`/`visited`/`session_status` changes. Reading `sync.session.get(id)` for every visited id is O(visited × map-lookup) per recompute; cheap. The `seen` Set prevents double-adding when a busy session is also in `visited` — correct.
- `context/session-tabs.tsx:165-172` (`move`) — uses `visible()` (memo) so navigation order matches what the user sees. `(currentIdx + direction + length) % length` correctly handles wrap-around in both directions. Clean.
- `context/session-tabs.tsx:188-202` (`closeActive`) — for ephemeral close, neighbour fallback uses `list[idx + 1] ?? list[idx - 1]` (no third fallback). If the user closes the only ephemeral tab while no pins exist, fallback is `undefined` → home. Consistent with `removeAndAdjust`. Good.
- `app.tsx:165-188` — `SessionTabsProvider` wraps `KeybindProvider` (so keybinds can reach the context via `useKeybind` registration on commands). Provider order looks correct; `SessionTabsProvider` reads `useSync`, `useRoute`, `useEvent`, all of which are higher in the tree. Good.
- `app.tsx:432-477` — four new commands. `session.tab.close` uses BOTH `enabled: ... && sessionTabs.isPinned(...)` AND `hidden: !sessionTabs.isPinned(...)`. The `hidden` flag also accounts for the route check, but `enabled` doesn't gate route as defensively. Minor: the close command is also useful for ephemeral tabs (where `closeActive` does work — drops from `visited`). The current gating only exposes it when pinned, which means users can't use the command palette to close an ephemeral tab. `closeActive` already supports this; loosen the predicate to `route.data.type === "session"` and let `closeActive` early-out internally.
- No tests in this PR. `session-tabs.tsx` has nontrivial logic (visible-list construction, delete handling, neighbour fallback) — at least a unit test for `removeAndAdjust` neighbour selection and the `visible` memo composition would be valuable. Existing TUI test patterns aren't visible from the diff but the project has prior TUI tests; follow the same pattern.

## Risk

Low. Self-contained feature, default-off (no pins until user pins). Persistence is JSON-on-disk in state dir, scoped per directory. Failure modes are graceful (read errors → empty list; write errors → swallowed by `void Filesystem.writeJson`). The main exposure is the missing startup-time stale-pin cleanup, which is cosmetic.

## Verdict

**merge-after-nits** — loosen `session.tab.close` enablement so it works for ephemeral tabs too (current gating is stricter than `closeActive`'s capability), add a startup pass that drops `pinned` ids no longer present in `sync.session`, and add at least one unit test for `removeAndAdjust` + the `visible` memo. Code quality is good and the SolidJS patterns (createStore, batch, createMemo) are used correctly.
