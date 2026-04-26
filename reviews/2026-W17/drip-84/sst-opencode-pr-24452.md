---
pr: 24452
repo: sst/opencode
title: "feat(tui): pinned session tabs in the right sidebar"
url: https://github.com/sst/opencode/pull/24452
head_sha: 780c717b12bdc3bc6304fea7a0b1fa5d2646c012
author: MrRobotoGit
verdict: needs-discussion
date: 2026-04-27
---

# sst/opencode #24452 — pinned session tabs in the right sidebar (TUI)

Closes #24451. Adds a SESSIONS panel at the top of the right sidebar
listing pinned sessions plus visited-this-run + currently-busy sessions
as ephemeral entries, with `<leader>p` to toggle pin, `<leader>w` to
close, `ctrl+up/down` to switch tabs. Pinned set persists per-directory
in `~/.local/state/opencode/session-tabs.json`. New context provider
`SessionTabsProvider` mounted inside `LocalProvider`.

## What the diff actually does

`packages/opencode/src/cli/cmd/tui/context/session-tabs.tsx:1-336` (new
214 LOC + 122 LOC component) — Solid context with reactive store
`{ ready, pinned: string[], visited: string[] }`. Persists `pinned` only;
`visited` is in-memory. `visible()` createMemo returns
`pinned ∪ ephemeral` where ephemeral = `(visited ∪ busyOrRetry) \ pinned`,
filtered through `sync.session.get(id)` so deleted sessions disappear.
Listens to `event.on("session.deleted", ...)` to drop pins automatically.

`packages/opencode/src/cli/cmd/tui/routes/session/session-tabs.tsx:1-104`
— renders the SESSIONS box. Active row gets `theme.backgroundElement`
background + accent left bar. Status glyph is `Spinner` when
busy, `↻` for retry, `●` for pinned-idle, `○` for ephemeral.

`packages/opencode/src/cli/cmd/tui/app.tsx:166-198` — wraps the existing
provider stack inside a new `SessionTabsProvider`. Important: it sits
*outside* `KeybindProvider` but *inside* `LocalProvider`/`SyncProvider`,
so it can read sync state but `KeybindProvider` can read its actions.

`packages/opencode/src/cli/cmd/tui/app.tsx:435-486` — adds four command
palette entries: `session.tab.pin`, `session.tab.close` (gated by
`isPinned(active)`), `session.tab.next/prev` (hidden when fewer than 2
pinned). Slash command `/pin` registered.

`packages/opencode/src/config/keybinds.ts:29-32` and
`packages/web/src/content/docs/keybinds.mdx:24-27` — register
`session_tab_pin = <leader>p`, `_close = <leader>w`,
`_next = ctrl+down`, `_prev = ctrl+up`. Doc page
`packages/web/src/content/docs/tui.mdx:190-204` adds `/pin` section.

`packages/opencode/src/cli/cmd/tui/routes/session/sidebar.tsx:8-50`
inserts `<SessionTabs />` + a 1-row separator above the existing
`<scrollbox>` content.

## Observations

1. **Keybind clash potential — `<leader>w`.** `session_tab_close` is
   bound to `<leader>w`. Look for existing `<leader>w` consumers; if any
   route also has a `<leader>w` action that's enabled outside session
   context (or even inside it), the new binding will shadow / collide.
   At minimum, the PR should grep `keybinds.ts` for `<leader>w` and
   confirm. Also `ctrl+up`/`ctrl+down` are commonly intercepted by
   terminal emulators (iTerm, kitty) for tab switching — that's the
   *intent* here but means many users will need terminal-side
   reconfig. Recommend defaulting to `<leader>[` / `<leader>]` and
   document that `ctrl+up/down` is opt-in.

2. **`session.tab.close` only works on pinned, not ephemeral active.**
   At app.tsx:451 the entry is gated `enabled: isPinned(route.data.sessionID)`.
   But `closeActive()` in session-tabs.tsx:308-323 *does* handle the
   ephemeral case (drops from `visited`, navigates to a neighbor). The
   command-palette gating means a user on an ephemeral tab can't reach
   `closeActive` via `<leader>w`. Either:
   - drop the `enabled` gate (always allow close, the function
     handles both cases), or
   - explicitly add a separate "Close ephemeral session tab" entry.
   The current behavior — close works on pinned only — leaves ephemeral
   tabs accumulating with no way to remove them mid-session.

3. **Persisted file is not migrated when sessions are deleted from
   another opencode process.** `event.on("session.deleted", ...)` only
   fires for deletes observed in *this* TUI run. If the user pins
   session X, exits, deletes X via the web UI, then relaunches TUI —
   `pinned` still contains X, and `visible()` filters it via
   `sync.session.get(id)` (line 267) so it doesn't render. Good, *but*
   the persisted file still has the stale id, so `byDirectory[cwd]`
   grows monotonically. Add a sweep in the load `.then(...)` block:
   filter `pinned` against `sync.session.get(id)` before setting and
   call `persist()` once. (Race: sync might not be ready yet at load
   time. If so, gate the sweep on a `sync.ready` effect.)

4. **`removeAndAdjust` fallback navigation has a subtle off-by-one.**
   ```ts
   const fallback = next[idx] ?? next[idx - 1] ?? next[0]
   ```
   After filtering out `id` from `before` to get `next`, `next[idx]`
   is what *was* at position `idx+1` (the right neighbor). That's
   intended. But `next[idx - 1]` is the original left neighbor, which
   exists iff `idx >= 1`. `next[0]` is fine as final fallback. Walk
   the cases:
   - close last pin: `idx = pinned.length - 1`, `next[idx]` = undefined
     (out of range, since `next.length = pinned.length - 1`),
     `next[idx-1]` = original `pinned[idx-1]` ✓.
   - close only pin: `idx = 0`, `next.length = 0`,
     `next[0] = next[-1] = next[0] = undefined` → navigate home ✓.
   - close middle pin: `next[idx]` = right neighbor ✓.
   OK, the logic is correct, but a 3-line comment naming each case
   would save a future maintainer the same walk-through.

5. **`writeState` is shared mutable state with no read lock.**
   `persist()` mutates `writeState.contents.byDirectory[directoryKey()]`
   then awaits `Filesystem.writeJson`. If two pin toggles fire
   back-to-back (faster than the file write resolves), both writes
   target the same object reference and the second is fine; but the
   filesystem writes themselves race. `Filesystem.writeJson` likely
   already serializes via tmp-rename, but worth confirming. If not,
   debounce with a 50-100ms timer.

6. **`directoryKey()` returns `sync.path.directory` but is not
   namespaced.** Two directories with the same basename in different
   parents are distinguished by absolute path (good). But on Windows,
   case-insensitive paths can mean `C:\Foo` and `c:\foo` get separate
   pin lists. Probably fine to ignore, but a one-line normalization
   (`.toLowerCase()` on win32) closes the gap.

7. **Command-palette `enabled` vs `hidden` inconsistency.** At
   `:451-456` the close entry uses *both* `enabled: ...` and
   `hidden: !isPinned ...`. Other entries use only one. If `hidden`
   alone is sufficient, drop `enabled`. If both are intentional (some
   downstream code respects only one), add a comment.

8. **No tests.** This is a 418/-17 LOC feature with persistence,
   cross-process state, event listeners, and four new keybinds. There
   are zero new tests. Minimum I'd ask for:
   - unit test for `SessionTabs` reducer logic (pin → unpin → busy
     event → deleted event) by mounting the provider in isolation;
   - golden test that `removeAndAdjust(id)` navigates to the expected
     neighbor for the three cases in observation 4;
   - persistence round-trip (write file, reload, assert pinned set).

## Verdict

**needs-discussion.** The feature is well-shaped and the diff is clean,
but four issues need maintainer input before merge: the `<leader>w`
binding, the close-ephemeral gap (obs 2), the stale-pin sweep on load
(obs 3), and the missing test coverage. Once those are resolved this is
a clean merge-after-nits.

## Banned-string scan

None.
