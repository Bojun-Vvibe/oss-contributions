# sst/opencode #24161 — feat: add /uptime slash command

**Link:** https://github.com/sst/opencode/pull/24161
**Tag:** small-feature, observability

## What it does

Closes #24160. Captures `Date.now()` once at module load in
`util/opencode-process.ts` (new top-level `PROCESS_START_TIME` const,
exposed via `processStartTime()` and added to `ensureProcessMetadata`'s
return). Surfaces it in two places:

1. The `GET /global/health` endpoint adds `startTime: number` to the
   200 response (Zod schema + SDK `types.gen.ts` regen).
2. A new TUI dialog (`dialog-uptime.tsx`) and slash command `/uptime`
   that renders `time + "up Nd Nh Nm Ns"` in the same centered-dialog
   style as `/status`.

## What it gets right

- Trivially scoped: one helper, one route field, one dialog, one command
  registration. No cross-cutting refactor.
- The start time is captured at **module load**, not on first call —
  so it reflects when the bundle was imported, which is close enough to
  "process start" for a long-lived agent and avoids the
  `process.uptime()` Node-only dependency. Works in Bun, Deno, and any
  embedding host.
- Server-side surface (`startTime` in `/global/health`) makes the same
  value available to non-TUI clients (web app, SDK consumers) without a
  separate endpoint.
- Duration formatting drops zero-valued leading units (no
  `0d 0h 0m 5s`) — small but nice.

## Concerns / risks

- **Module-load semantics under HMR / repeat imports** are not what the
  user expects. In dev with hot module reload, `PROCESS_START_TIME`
  resets every time `opencode-process.ts` is re-imported. The dialog
  will then report wall-clock-since-last-HMR rather than process
  uptime. For a dev-only convenience this is fine; for production it's
  a non-issue (no HMR), but worth a `// captured at module load — not
  process start under HMR` comment.
- Multi-process layouts (worker subprocesses calling
  `ensureProcessMetadata("worker")`) will now report the **worker's**
  load time as `startTime` in the metadata payload, not the main
  process's. If anything downstream merges these, the value becomes
  ambiguous. Not exercised by this PR but worth keeping in mind.
- The dialog computes `now - processStartTime()` once at render. There's
  no tick — open the dialog, leave it open for an hour, the displayed
  uptime stays frozen. A trivial `setInterval(1000)` + `useState` would
  fix it; or document that re-opening refreshes.
- `/global/health` previously had a tight `{ healthy, version }` shape.
  Adding a required `startTime` is a contract widening; downstream
  consumers using strict zod parse (`.strict()`) elsewhere would now
  reject responses from older servers. SDK type bump handles the
  positive direction; mixed-version deployments may surface this.

## Suggestion

Either tick the dialog (1s `setInterval` + cleanup on unmount) so the
displayed uptime advances while open, or rename the dialog to
`/uptime-snapshot` to make the static-at-render behavior explicit. The
current `up 3h 1m 45s` text strongly implies live counting.
