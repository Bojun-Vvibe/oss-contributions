# sst/opencode #24541 — fix(tui): handle background sync rejection in sync.tsx

- Author: alfredocristofano (Alfredo Cristofano)
- Head SHA: `2837bd8a0e3a48686bd2693f5fd933787fafc7aa`
- Single file: `packages/opencode/src/cli/cmd/tui/context/sync.tsx` (+13 / −5)

## Specifics

- At `sync.tsx:421-451`, the inner `Promise.all([...])` was previously fired off as `void Promise.all([...]).then(...)` — a fire-and-forget block with **no `.catch`**, so any rejection from `sessionListPromise`, `consoleStatePromise`, `sdk.client.command.list`, `sdk.client.provider.auth`, `sdk.client.vcs.get`, or `project.workspace.sync()` would surface as an unhandled promise rejection and (depending on the runtime) either crash the TUI process or get swallowed silently while the UI sat stuck in the `partial` state forever.
- Fix returns the `Promise.all(...)` from the outer `.then(...)` so the outer chain takes ownership of the result, then chains `.then(() => setStore("status", "complete"))` and `.catch((e) => { Log.Default.error(...); setStore("status", "complete") })`. Critical bit: `setStore("status", "complete")` is called on **both** branches so the spinner unblocks even when sync partially failed — matches the `if (store.status !== "complete") setStore("status", "partial")` invariant set just above.
- Logged fields `error`, `name`, `stack` use the standard `e instanceof Error ? ... : ...` guard pattern already used by the sibling `.catch` at `sync.tsx:445+` for `tui bootstrap failed`. Consistent with the file's existing style.
- The `void` keyword removal is necessary because the promise is now consumed by `return` — leaving `void` would have been a TS error or at minimum dead syntax.

## Concerns

- One subtle behaviour change: previously the outer `.then(() => {...})` callback returned `undefined` (the `void` expression), so the outer chain resolved as soon as the inner block was kicked off. Now the outer chain awaits the inner `Promise.all` before resolving. If anything upstream of this code awaits the outer promise and depends on early return, this introduces a latency change. From the diff context this looks unlikely (it's inside an async bootstrap effect that is itself fire-and-forget at the call site), but the PR description doesn't call this out.
- Marking status `complete` on failure means the user sees no signal that some store slices are stale. Acceptable trade-off vs. infinite spinner, but a future improvement could surface a `partial-failed` status.

## Verdict

`merge-as-is` — strict bug fix with no scope creep, idiomatic error logging matching the file's existing pattern, no test added but the surface is trivial enough that visual regression in the TUI bootstrap path is the realistic test.

