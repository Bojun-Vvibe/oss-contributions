# sst/opencode PR #24877 — fix(core): run sessions with the same directory they were created with

- Repo: sst/opencode
- PR: https://github.com/sst/opencode/pull/24877
- Head SHA: `320f37917f11052cbb3d768031625ba171f86772`
- Author: jlongster
- Size: +145 / −20, 3 files

## Context

Until this PR, opencode's HTTP layer resolved a session's effective working
directory at request time from `?directory=` on the URL, even on routes that
target an existing session by id (e.g. `POST /session/:id/shell`,
`POST /session/:id/fork`, `GET /session/:id`). That overload meant the same
session would behave differently depending on which client window's CWD
issued the request, with predictable bug surfaces: a `pwd` shell turn that
returned the wrong root, syncs that emitted under the wrong `event.directory`
and so were filtered out by clients listening on the originating workspace,
and forked sessions silently re-rooting against whatever the forking client
was looking at.

## What the diff actually does

Two production sites and one test file:

1. `packages/opencode/src/server/workspace.ts:39-71` — `getSessionWorkspace`
   gets renamed to `getSession` and returns the whole session record (was:
   just `session?.workspaceID`). The middleware then uses
   `session.directory` to wrap the downstream handler in
   `Instance.provide({ directory: session.directory, ... })` *even when the
   workspace-routing branch falls through* (no workspace id, console route,
   or `OPENCODE_WORKSPACE_ID` flag set). Net: any request that resolves a
   session by id runs the handler under that session's recorded directory,
   regardless of `?directory=` on the URL.

2. `packages/opencode/src/cli/cmd/tui/context/event.ts:12-26` — the TUI's
   global event handler used to gate `handler(event.payload)` on a tri-state
   match (`directory === "global"`, then `event.workspace ===
   project.workspace.current()`, then `event.directory ===
   project.instance.directory()`). All three gates are deleted — the TUI now
   trusts the server to filter and just forwards every payload. This is the
   load-bearing change for the user-visible bug: previously the server emit
   carried the *requesting* client's directory, which the TUI then filtered
   *out* because it didn't match the workspace it was attached to.

3. New test at `packages/opencode/test/server/session-routes-directory.test.ts`
   exercises the bug shape end-to-end: creates a session in `sessionDir`,
   then issues `GET /session/:id`, `POST /session/:id/fork`, and
   `POST /session/:id/shell` with `?directory=requestDir.path`, and asserts
   that `event.directory` on the resulting `session.created.1` /
   `message.updated.1` / `message.part.updated.1` sync events all carry
   `sessionDir.path`, not `requestDir.path`. The non-session route
   (`/file/content`) is asserted to still honor `requestDir.path` — exactly
   the discriminating assertion that pins both halves of the fix.

## Risks and gaps

1. **Stale `event.workspace` filter removal is the right answer for the
   reported bug, but it widens the trust boundary**. If any code path emits
   a global event whose `directory` doesn't match what the TUI is showing
   *and* the user has multiple workspaces open in separate panes, the TUI
   now can't filter them client-side. The PR description should explicitly
   call out that *all* directory/workspace filtering for sync events now
   happens server-side, so any future server-emit needs to set
   `event.directory` correctly or the wrong client will render it.

2. **The `Instance.provide({ directory: session.directory, ... })` wrap
   inside the `if (!workspaceID || ...)` branch is a behavioral change for
   the console route**. Previously `/console/*` requests would fall through
   to `next()` with the request's CWD; now if the URL path also happens to
   carry a `sessionID`, the console handler runs under the session's
   directory instead. Probably fine for the console use case, but worth a
   regression test that pins which directory the console sees when given a
   session id.

3. **`session?.directory` is read without a `null`-coalesce or schema
   guarantee**. If `Session.Service.use(...).get(id)` ever returns a session
   record whose `directory` is empty / unset (legacy DB rows from before the
   field was required), the wrap will pass `directory: undefined` into
   `Instance.provide`, which presumably falls back to process.cwd or
   crashes. A one-line `if (session.directory)` guard would harden this
   against legacy data.

4. **The TUI's deleted three-tier filter previously also handled the
   `event.directory === "global"` shortcut**. After the deletion the TUI
   handles every event regardless of whether it was tagged "global" — which
   is the *intended* change, but means any handler that registered through
   `useEvent()` is now load-bearing for noise control and must filter the
   payload itself. Worth grepping `useEvent(` to confirm no handler relied
   on the gate for correctness rather than performance.

## Suggestions

- Add a CHANGELOG / release-note line covering the trust-boundary widening:
  "Sync event filtering by directory/workspace is now exclusively
  server-side. Custom TUI handlers must filter payloads themselves."
- Add a defensive `if (session.directory)` guard before the `Instance.provide`
  wrap so legacy session rows without `directory` don't crash the handler.
- Add a one-line console-route test pinning that `/console/...?sessionID=X`
  does or does not run under `X.directory` (whichever is intended).
- Grep `useEvent(` call sites to confirm none of them depended on the
  deleted `event.directory === project.instance.directory()` gate for
  correctness rather than as a perf shortcut.

## Verdict

**merge-after-nits** — the fix correctly identifies that session id is the
right key for directory resolution on session routes, and the test
discriminates the bug shape properly (asserts both that session routes use
the recorded dir and that non-session routes still honor the request dir).
The trust-boundary widening on the TUI side deserves a release note but
isn't a blocker.
