# Review: sst/opencode#24877 — fix(core): run sessions with the same directory they were created with

- **Author**: jlongster
- **Head SHA**: `320f37917f11052cbb3d768031625ba171f86772`
- **Diff**: +145 / −20 across 3 files (`packages/opencode/src/cli/cmd/tui/context/event.ts`, `packages/opencode/src/server/workspace.ts`, `packages/opencode/test/server/session-routes-directory.test.ts`)
- **Draft**: no

## What it does

Existing behavior: any HTTP route on the Hono app extracted a `directory` query param, derived an `Instance` from it, and ran the handler in that instance. For session-scoped routes (`/session/:id`, `.../fork`, `.../shell`) this meant a session created against `/projA` would emit its events under whatever cwd the *current* request happened to attach — typically the wrong one when a TUI on `/projB` continues an old session. Sync events therefore landed on the wrong workspace bus and the session appeared to silently jump cwds.

The fix in `server/workspace.ts:42-71` renames `getSessionWorkspace` → `getSession` (now returning the whole session record, not just `workspaceID`) and, in the no-`workspaceID` branch, when a session *was* found by the URL's `id`, wraps `next()` in `Instance.provide({ directory: session.directory, init: () => AppRuntime.runPromise(InstanceBootstrap), fn: ... })`. The non-session path is unchanged. A 130-line full-Hono integration test (`test/server/session-routes-directory.test.ts`) creates two `tmpdir` worktrees, posts a session against the first, then drives `/path`, `/file/content`, `/session/:id`, `/session/:id/fork`, `/session/:id/shell` while pointing the request `directory` at the *second*, and asserts (a) `marker.txt` reads `request-directory` (non-session route still uses request dir) and (b) every emitted `session.created.1` / `message.updated.1` / `message.part.updated.1` event carries `directory: sessionDir.path`.

The companion deletion in `cli/cmd/tui/context/event.ts:12-30` rips out all the in-TUI directory/workspace event filtering and just calls `handler(event.payload)` unconditionally, on the theory that now that the *server* tags events with the right `directory`, the client-side filtering was both redundant and the proximate cause of dropped events when the server emitted under the "wrong" dir.

## Concerns

1. **TUI filter removal is a much wider blast radius than the title implies.** Pre-PR, `useEvent` had three guards: a `"global"` short-circuit, a workspace-id match, and a request-dir match. Post-PR, every event delivered to the bus runs every handler. That's correct *if and only if* the bus subscriber is already scoped per-instance, but the diff doesn't show that it is — it just deletes the guards. If two TUI panes (or a TUI plus a daemon process) share a `GlobalBus`, the wrong pane will now process foreign events. The PR test only exercises the *server* path; it does not assert anything about TUI handler dispatch. A second test like "two `useEvent` subscribers on different cwds receive only their own events" would close the gap, or at minimum a comment in `event.ts:12-15` explaining why the filtering can be safely dropped now.

2. **`Instance.provide` is now created inline per request without a clear caching story.** The `init: () => AppRuntime.runPromise(InstanceBootstrap)` runs full bootstrap on every session-route hit when no workspaceID is set. That's the behavior on the existing branches too, but the existing branches typically had a workspaceID and so took the cached `WorkspaceRouterMiddleware` path — this PR widens the set of hits that fall through to the slow path. Worth confirming there's an instance cache keyed by `session.directory` somewhere downstream of `Instance.provide`, or this becomes a per-request `bun install`/`tsserver`-warmup tax.

3. **`session.directory` provenance is implicit.** The fix relies on `session.directory` being authoritative and immutable since session create. That's almost certainly true today, but the test doesn't pin it (e.g. there's no "fork emits events under parent's `session.directory` even if `fork` was invoked from a third dir" assertion — only that fork emits under the original `sessionDir.path`, which doesn't distinguish "uses fork's request dir which happens to be sessionDir" from "uses session.directory by lookup"). Since the test passes `requestDir.path` as the request dir for the fork call, this is actually well-pinned for fork — apologies, partial retract — but the per-event `directory` field equality is the only signal; an additional assertion that `Instance.directory()` inside the handler equals `sessionDir.path` would make the contract harder to regress accidentally.

4. **`/console` and `Flag.OPENCODE_WORKSPACE_ID` paths still skip the new branch.** `workspace.ts:60-68` only enters the new `Instance.provide` block when neither `/console` nor the global flag are set. If a session lookup succeeds *and* `Flag.OPENCODE_WORKSPACE_ID` is set, the request falls through to the bare `next()` — likely the right behavior (operator override) but not stated.

## Verdict

**merge-after-nits** — server fix is well-targeted and the integration test is real (full Hono app, real session create, real shell turn). The TUI-side mass removal of `useEvent` filtering is the load-bearing risk and deserves a one-paragraph rationale comment near `event.ts:12` plus a test that pins "two cwd-distinct subscribers get only their own events," because if that invariant ever shifts the regression will look like cross-workspace event leakage rather than a missing directory tag.

