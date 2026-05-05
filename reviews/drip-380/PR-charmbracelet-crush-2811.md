# charmbracelet/crush PR #2811 — feat: add --all and --crawl-dir modes to stats subcommand

- URL: https://github.com/charmbracelet/crush/pull/2811
- Head SHA: `8548ed7545f17ee1d708bb57366653df5c391234`
- Size: +565 / -54

## Summary

Extends `crush stats` from "single-project current-CWD" to two cross-project aggregation modes: `--crawl-dir <path>` (recursively walks for `.crush/crush.db`) and `--all` (reads `projects.json` and aggregates each project's DB). Adds a parallel read-only DB connector, an in-memory aggregator over all per-axis maps (daily / model / hourly / day-of-week / tool / heatmap), a per-project breakdown table in the rendered HTML, and a UTC→local timezone shift in the heatmap.

## Specific findings

### Backend / Go

- `internal/cmd/stats.go:53-56` — `init()` registers two new flags. Flag names (`--crawl-dir`, `--all`) and descriptions are accurate.
- `internal/cmd/stats.go:139-152` — flag dispatch is mutually exclusive via `switch case ... default`. **Concern**: nothing rejects `--all --crawl-dir=foo` together; the switch silently picks `--crawl-dir` (first case). Should either be enforced via cobra's `MarkFlagsMutuallyExclusive` or documented.
- `internal/cmd/stats.go:243-275` — `crawlForStats` walks `rootDir`, skipping common dirs via `shouldSkipDir`. The matcher at `:259-263` looks specifically for `crush.db` files whose parent dir is named `.crush` — correct hardcoding of the storage convention.
- `internal/cmd/stats.go:279-307` — `shouldSkipDir` denylist is reasonable but **`Library`, `Applications`, `System` will skip those dirs even on Linux/Windows where they may be legitimate user paths**. Probably fine in practice (user wouldn't store crush projects there), but worth a comment noting the macOS-isms.
- `internal/cmd/stats.go:328-371` — `gatherStatsFromDBPaths` runs DB reads concurrently with a `sem := make(chan struct{}, 10)` semaphore. `wg.Wait()` then aggregates. **Concern**: errors from individual DBs are logged to stderr (`:354`, `:360`) and silently dropped — a corrupt DB will not fail the whole run. Right call for `--all` UX, but the user has no way to see "5 of 12 projects skipped". Recommend a final summary line like `warn: skipped N project(s)` before `return results, nil`.
- `internal/cmd/stats.go:375-487` — `mergeStats` is the load-bearing aggregation. Per-axis map keys:
  - `dailyUsageMap[d.Day]` — keyed on the `YYYY-MM-DD` string. Correct for summing across projects on the same day.
  - `modelUsageMap[m.Model + "|" + m.Provider]` — `|` separator is fine; collision risk only if model name contains `|`, unlikely.
  - `hourlyUsageMap[h.Hour]` and `dayOfWeekMap[d.DayOfWeek]` — int keys, correct.
  - `toolUsageMap[strings.ToLower(t.ToolName)]` — normalizes case across projects. Right choice.
  - `heatmapMap[fmt.Sprintf("%d-%d", h.DayOfWeek, h.Hour)]` — composite key, fine.
- `internal/cmd/stats.go:464-470` — average response time computed as token-count-weighted: `totalResponseTimeMs += s.AvgResponseTimeMs * float64(s.Total.TotalMessages)` then divide by sum of messages. **Subtle issue**: this is correct only if each per-project `AvgResponseTimeMs` is itself an unweighted message-mean. If gather computes it as a session-mean or excludes some message classes, the recombination is wrong. Worth a 1-line comment pinning the assumption.
- `internal/cmd/stats.go:487-493` — final sort applies only to `UsageByModel` and `ToolUsage`. Other axes (daily, hourly, day-of-week, heatmap) ship out unsorted, relying on the JS to sort them. Fine but inconsistent.
- `internal/cmd/stats.go:651` — `GeneratedAt: time.Now().UTC()` (changed from `time.Now()`). Required for the new client-side UTC→local conversion to be correct.
- `internal/db/connect.go:78-99` — `ConnectReadOnly`: opens via the new `openDBReadOnly` helper, sets `MaxOpenConns(1)`, pings. Good — no migration runs, can't accidentally write to another project's DB.
- `internal/db/connect_modernc.go:13-26` — `mode=ro` + `_txlock=immediate` for modernc driver.
- `internal/db/connect_ncruces.go:13-21` — same for ncruces driver. Both drivers covered, consistent.

### Frontend

- `internal/cmd/stats/index.js:33-35` — `formatCost` now switches to `Math.round(n).toLocaleString()` at $1000+. Sensible.
- `internal/cmd/stats/index.js:37-46` — `formatDate` parses date strings as UTC then renders local. Correct.
- `internal/cmd/stats/index.js:144-145` — `utcOffsetHours = new Date().getTimezoneOffset() / -60`. **Issue**: this uses *today's* offset to retroactively shift historical heatmap data. For users who cross DST during the data window, some bins shift by 1h relative to others. Acceptable approximation for a stats view but worth a tooltip note.
- `internal/cmd/stats/index.js:163-176` — heatmap day-rollover logic: when `h.hour + utcOffsetHours < 0` shift day backward, when `>= 24` shift forward. Math is correct.
- `internal/cmd/stats/index.js:382-436` — per-project breakdown table is gated on `projectStats.length > 1`. Single-project (default) mode skips it cleanly. `displayPath.split("/").pop()` for project name + `dirPath` for the rest — fine for POSIX paths, will look odd on Windows (`\` separator). Crush already runs cross-platform, so worth a note.

## Concerns / nits summary

1. `--all` + `--crawl-dir` conflict not enforced.
2. Per-project skip count not surfaced to the user.
3. `AvgResponseTimeMs` weighted-mean assumption needs a comment.
4. Heatmap UTC→local shift uses current-day's offset for all historical data (DST drift).
5. Windows path split in JS for project breakdown table.
6. macOS-named skip dirs (`Library`/`Applications`/`System`) not noted as such.

## Verdict

`merge-after-nits` — substantial and well-structured feature; backend is correct; main concerns are UX edge cases (conflict-flag handling, silent skip count) and the documented assumptions around weighted-mean response time. Nothing blocking.
