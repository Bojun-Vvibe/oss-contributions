# sst/opencode#24826 — feat(opencode): show model provider quota in prompt metrics

- **Repo**: sst/opencode
- **PR**: [#24826](https://github.com/sst/opencode/pull/24826)
- **Author**: (per PR header)
- **Head SHA**: `46e1a918a2c436a5b28cdea3e3998a0b5d7e09ac`
- **Reviewed**: 2026-04-29
- **Verdict**: `merge-after-nits`

## Context

The TUI's prompt-metrics row currently shows `context · cost`. This PR
adds a third optional segment: a per-provider quota indicator
(remaining-percent bar, optional fetch timestamp), wired up first for
the OpenAI Codex provider via its OAuth session and a "best-effort"
private console endpoint. The new segment renders below the chat box
in the same row as context/cost, so the existing footer/version line
isn't disturbed.

## What changed

### `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx`

```tsx
const quota = createMemo(() => {
  return formatCodexQuotaMetrics(sync.data.console_state.codexQuota, terminal().width)
})
const metrics = createMemo(() => {
  const parts = [usage()?.context, quota(), usage()?.cost].filter(Boolean)
  if (parts.length === 0) return
  return parts.join(" · ")
})
```

The `<Match when={metrics()}>` replaces the previous `<Match when={usage()}>`
gate. `terminal().width` now drives quota layout via a new
`useTerminalDimensions` hook subscription. Field ordering inside the
joined metrics is fixed: `context · quota · cost`.

### `packages/opencode/src/cli/cmd/tui/component/prompt/metrics.ts` (new)

Pure formatter — no I/O, no async. Three exported helpers:

- `formatCodexQuotaFetchedAt(timestamp, now)` — `today@HHhMMmSSs` /
  `1d ago@HHhMMmSSs` style label with NaN-guard, `pad(2,'0')` on
  H/M/S, midnight-aligned day-diff via local-tz `Date(y,m,d).getTime()`.
- `formatQuotaBar(percent, width)` — Unicode `█` / `░` block bar.
  `clampPercent` folds `NaN`/`±Infinity` to 0, then clamps 0..100.
- `formatCodexQuotaMetrics(snapshot, terminalWidth, now)` — composes
  `5h …`, `wk …`, optional `⟳ <fetchedAt>` based on a width-tiered
  layout map: `≥200` → bar+timestamp; `≥120` → narrower bar+timestamp;
  `≥90` → percent+timestamp; otherwise → bare `5h N% · wk N%`. Returns
  `undefined` when no `fiveHour`/`weekly` data is present, so the
  consumer's `.filter(Boolean)` cleanly excludes it.

### `packages/console/resource/resource.node.ts`

Two unrelated TS-narrowing tweaks: the `client.kv.namespaces.bulkGet`
and `keys.list` `.then` callbacks now type their `result` parameter as
`Awaited<ReturnType<typeof …>>`. Looks like sympathy fixes for stricter
TS settings — separable from the feature, but not worth blocking the PR
over.

## Design call-outs (good)

- **Pure formatter, async data flow**. The new file is 100% pure — all
  reactivity flows through `sync.data.console_state.codexQuota`, which
  is presumably fed by the existing console-state pipeline. Test surface
  is unit-testable without mocking SolidJS or the terminal.
- **Width-tier layout map**. Better than runtime fitting algorithms for
  TUIs — predictable, easy to test, easy to tune by hand.
- **Three-tier graceful degradation**. The `barWidth + timestamp →
  no-bar + timestamp → bare percent` ladder means the indicator stays
  legible at any terminal width without any wrap/truncation logic.

## Things I'd want changed

1. **The "Codex usage source is ChatGPT's private
   `/backend-api/wham/usage` endpoint" disclaimer in the PR body
   should land in a code comment** at the call site, not just in the
   PR description. Future maintainers staring at a sudden 401/403
   from that endpoint won't go reading the PR; they need the
   "this is intentionally best-effort, undocumented, no SLA" note
   inline.

2. **Day-diff edge case across DST**. `Date(y,m,d).getTime()` is
   local-tz; on a DST transition day, the `nowStart - dateStart`
   division by `86_400_000` can be off by an hour, which can flip
   the floor result by 1 in a narrow window around midnight on
   spring-forward days. The cosmetic impact is "today" vs "yesterday"
   for a few hours. Not blocking — fix it later by switching to
   `Math.round` or by reading the diff in days from a timezone-aware
   helper.

3. **Tests for the new file aren't in the diff hunks I'm seeing**
   (the PR body says `test/cli/cmd/tui/prompt-metrics.test.ts` was run
   — assume it exists and was added). The file is structured for
   unit testing but I'd want the layout-tier ladder asserted
   explicitly: e.g. `formatCodexQuotaMetrics(snapshot, 89)` returns no
   timestamp; `width=90` returns timestamp; `width=119` no bar;
   `width=120` 5-wide bar; `width=200` 10-wide bar. These boundary
   tests prevent the next refactor from silently shifting the
   layout points.

4. **Console-state field naming**. `codexQuota.fiveHour` /
   `codexQuota.weekly` are baked into the TUI directly. If a future
   provider exposes different windows (daily, monthly), the quota
   formatter will need restructuring. Worth a comment that the
   schema is provider-specific even though the formatter pretends
   to be generic.

## Risk analysis

- **Backwards-compat**: when `codexQuota` is absent (any non-Codex
  provider, or Codex with no auth), `quota()` returns `undefined` and
  the existing `context · cost` join is unchanged. Safe.
- **Performance**: formatter is O(width) per render via the bar
  `repeat`. Negligible.
- **Privacy/leak surface**: the quota numbers themselves are scoped
  to the local user's account. The new endpoint call is described as
  "best-effort, non-blocking" — assuming that means "don't surface
  errors, don't retry-storm". Worth one sentence in code confirming
  that.

## Verdict

`merge-after-nits`. Solid feature wiring with a clean separation
between pure formatter and reactive surface. The endpoint-stability
disclaimer needs to migrate from the PR body into a comment, and the
layout-tier boundaries deserve direct test coverage. Neither is
blocking on its own.

## What I learned

For TUI metrics surfaces that need to fit varying terminal widths, a
fixed-tier layout map (a small table of `width >= N → layout`) is
both more predictable and more testable than dynamic-fitting
algorithms. The trade-off is that you have to hand-tune the
breakpoints — but those tunings end up being the kind of thing you
*want* to be visible and version-controlled, not the output of an
opaque packing algorithm.
