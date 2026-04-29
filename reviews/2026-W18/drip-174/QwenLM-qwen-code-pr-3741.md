# QwenLM/qwen-code #3741 — feat(cli): add MCP health pill to footer

- **PR:** https://github.com/QwenLM/qwen-code/pull/3741
- **Title:** `feat(cli): add MCP health pill to footer`
- **Author:** wenshao (Shaojin Wen)
- **Head SHA:** d9a3d9005881e3852b0a945bb0c86d65b39b114e
- **Files changed:** 4 (`Footer.tsx` +2, new `MCPHealthPill.tsx` +46, new `MCPHealthPill.test.tsx` +64, new `useMCPHealth.ts` +70), **+182 / 0**
- **Verdict:** `merge-as-is`

## What it does

Adds a tiny inline pill to the Footer's left-bottom region that
displays `· N MCP(s) offline` (warning color) whenever any configured
MCP server is in `DISCONNECTED` state. Hidden when no MCPs are
configured or all are healthy. Auto-clears once the existing 30s health
check reconnects.

Three new files:

1. **`useMCPHealth.ts:1-70`** — subscribes to the existing
   `mcp-client` module-level listener API (`addMCPStatusChangeListener`
   /`removeMCPStatusChangeListener` / `getAllMCPServerStatuses`) and
   exposes raw counts (`totalCount`, `disconnectedCount`,
   `connectingCount`, `connectedCount`) as a snapshot object. Returns
   counts not a formatted label so future surfaces (boot screen,
   tooltips) can derive their own presentation.
2. **`MCPHealthPill.tsx:1-46`** — renders the formatted label via the
   exported-for-testability `getPillLabel(snapshot)` function. Hidden
   path is `if (!label) return null`, single-vs-plural handled by
   `disconnectedCount === 1 ? '' : 's'`.
3. **`Footer.tsx:181`** — single new line `<MCPHealthPill />`
   immediately after `<BackgroundTasksPill />` in the same flexbox
   region.
4. **`MCPHealthPill.test.tsx:1-64`** — 6 vitest cases covering empty,
   all-connected, all-connecting (suppressed), 1 offline (singular),
   3 offline (plural), and mixed (counts only disconnected). All
   exercised via the pure `getPillLabel` function — no renderer needed.

## Why this is correct

The PR body's design rationale is unusually thorough and mostly
correct:

- The "why a parallel pill, not extending the Background tasks
  `DialogEntry` registry" explanation calls out the actual category
  error: tasks have a *terminal status* contract
  (`running → {completed, failed, cancelled}` + `notified`), MCPs
  have a *connection-state cycle* (`disconnected ↔ connecting ↔
  connected`) with auto-reconnect. Forcing MCPs into the registry
  would dilute the framework's invariants.
- Suppressing `connecting` from the pill (per `getPillLabel` line 24
  and the test at lines 39-43) is the right call — boot/reconnect
  transitions would otherwise make the pill flicker. The state worth
  surfacing is the one that doesn't recover on its own:
  `DISCONNECTED`.
- `useMCPHealth` returns the raw counts, not a string. That's the
  right shape — keeps the hook reusable for future surfaces and
  makes the pill's presentation logic isolatable in `getPillLabel`,
  which is where the test surface lives.
- The "v1 visual indicator only, focus chain deferred" framing is
  honest about scope and defers the focus-chain abstraction problem
  (which is genuinely larger than it looks once a third pill lands)
  to its own PR.

## What's good

- **Tests are pure and exhaustive** for the formatting layer. 6 cases
  cover the cardinality boundaries (0/1/N) and the suppression rule
  (`connecting` doesn't count). The hook itself is presumably tested
  via the integration of the listener API in mcp-client tests, so
  not duplicating that surface here is the right call.
- **`useMCPHealth.ts:50-58`** correctly does the resync-after-listener
  attach pattern: `setServers(new Map(getAllMCPServerStatuses()))`
  inside the `useEffect` after `addMCPStatusChangeListener(listener)`.
  This handles the edge case where the registry transitions between
  the initial snapshot capture (line 47) and listener attachment
  (line 50). Without this, a server that flipped status mid-mount
  would be permanently stale.
- **Clean teardown** at line 51 returns
  `() => removeMCPStatusChangeListener(listener)` from the effect, so
  the hook is correct under StrictMode double-invocation and
  hot-reload.
- **Footer integration is one line** (`Footer.tsx:181`) and reads
  exactly like the adjacent `<BackgroundTasksPill />` — no special
  positioning, no flex magic, just lexical adjacency. The
  established pattern absorbs the new consumer cleanly.
- **`getPillLabel` is exported separately for testability** rather
  than being inlined into the component — the PR clearly understood
  that pure functions are easier to test than components that need
  ink renderers and listener mocks.
- **No public API additions** beyond what already existed in
  `mcp-client`. The hook is a pure consumer, no two-way coupling.

## Nits / risks

The PR body already volunteers most of the obvious follow-ups
(focus-chain integration, optional `connecting` after stuck-threshold).
A few small things worth thinking about post-merge but not blocking:

1. **State updates are coarse-grained.** `useMCPHealth` re-creates the
   entire `Map<string, MCPServerStatus>` on every status change
   (`useMCPHealth.ts:42-45`), which forces a re-render of every
   consumer even when only one server's status changed and the
   *aggregate* counts didn't move (e.g. `connecting → connected`
   doesn't change `disconnectedCount` but still re-renders the pill).
   Tolerable in v1 because the only consumer is one Footer component
   and status changes are rare. If a second consumer lands, consider
   memoizing on `(disconnectedCount, totalCount)`.
2. **`connectedCount` and `connectingCount` are computed but never
   surfaced** in the v1 pill. They're in the snapshot for future
   consumers (boot screen, tooltips) per the design notes — fine.
   Worth a one-line `// not used by MCPHealthPill in v1; exposed for
   future consumers (boot screen, tooltips)` comment in
   `useMCPHealth.ts:24-25` so the next reader doesn't try to
   `// eslint-disable-next-line @typescript-eslint/no-unused-vars` it.
3. **No screenshot in PR body.** A real terminal capture showing the
   pill rendering in warning color next to BackgroundTasksPill would
   help reviewers who don't want to spin up a deliberately-bad MCP
   to visualize. Not blocking — the test cases nail the label, and
   the styling delegates to `theme.status.warning` which is already
   the established warning color.
4. **`getAllMCPServerStatuses()` shape contract.** The hook assumes
   it returns something `Map`-constructible (an iterable of
   `[name, status]` tuples). If the upstream API ever changes to
   return a plain object, the hook silently breaks at
   `useMCPHealth.ts:38, 47, 50`. A type assertion in the import or a
   tiny `assertIsMapLike` adapter would harden this.

## Verdict

`merge-as-is` — exemplary feature PR. Minimal blast radius (one
Footer line + 3 small new files), correct hook lifecycle, exhaustive
tests on the pure formatting layer, well-justified architectural
decision (parallel pill, not registry-extension), explicit and honest
scope (v1 indicator-only, focus-chain deferred), and a real UX
problem solved (silent MCP failures invisible until `/mcp`). The nits
are post-merge polish, not merge blockers.
