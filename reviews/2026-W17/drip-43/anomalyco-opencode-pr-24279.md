# anomalyco/opencode PR #24279 — fix(app): align usage chart with local timezone

- **URL:** https://github.com/anomalyco/opencode/pull/24279
- **Author:** @MrMushrooooom
- **State:** OPEN (target `dev`)
- **Head SHA:** `6177759ca8e9d560b3cf9c6d601c8e261842a733`
- **File:** `packages/console/app/src/routes/workspace/[id]/usage/graph-section.tsx`

## Summary of change

Fixes a long-standing bug in the usage graph where users in
non-UTC timezones could see daily costs attributed to the wrong
day (or dropped entirely for boundary hours). The change moves
bucket-aggregation from server-side `DATE(timeCreated)` (UTC date)
to a client-side hour bucket (`FLOOR(unix_timestamp/3600)`) that
the browser then re-bins into local-timezone days.

## Findings against the diff

- **Server query (L29–58):**
  ```ts
  const startDate = new Date(Date.UTC(year, month, 1))
  startDate.setUTCDate(startDate.getUTCDate() - 1)   // -1 day buffer
  const endDate = new Date(Date.UTC(year, month + 1, 1))
  endDate.setUTCDate(endDate.getUTCDate() + 1)       // +1 day buffer
  ```
  Correct: fetching ±1 UTC day around the target month covers the
  ≤24h timezone offset window so a user in UTC+14 (Kiribati) or
  UTC−12 still sees boundary hours correctly. `hourBucket =
  FLOOR(UNIX_TIMESTAMP / 3600)` gives a deterministic integer per
  UTC hour — much better than carrying a string date.
- **Client formatter (L130–137):**
  ```ts
  function formatDateLabel(dateStr: string): string {
    const [year, month, day] = dateStr.split("-").map(Number)
    const date = new Date(year, month - 1, day)
    return `${date.toLocaleDateString(undefined, { month: "short" })} ${day.toString().padStart(2, "0")}`
  }
  ```
  Removed the `.getUTCDate()` confusion of the old version (which
  built a local-time `Date` then read `getUTCDate()` — a known
  off-by-one source). Uses the parsed `day` directly. ✅
- **`formatDateKey(date)`** (L137): builds `YYYY-MM-DD` from
  *local* `getFullYear/getMonth/getDate`. Used both to produce the
  full-month label set and to re-bucket each `hourBucket` row.
  The two places use the *same* function, so labels and rows can't
  drift apart. ✅
- **`getUsageForMonth` (L189–195):** filters rows whose
  `formatDateKey(new Date(hourBucket*3600*1000))` is *not* in the
  current month's date set — that's how the ±1 day fetch buffer
  gets discarded client-side. Clean.
- **`chartConfig` (L243–249):** the per-row dateKey is recomputed
  with the same `formatDateKey(new Date(hourBucket * 3600 * 1000))`,
  ensuring rows attribute to the local day a user actually
  experienced. Replaces the old `row.date` (server-UTC string).
- **L300 `if (datasets.length === 0) return null`:** new guard so
  an empty filtered set (e.g. user picked a key that has no usage
  in that month) doesn't render an empty `<canvas>`.
- **Performance note:** the row volume goes up because aggregation
  is now per-(hour, model, key, plan) instead of per-(day, …).
  For a 31-day month with 5 models × 3 keys × 3 plans, worst case
  is `744 × 45 ≈ 33k rows` — still trivially small for the
  WASM-side chart, and the `groupBy` clause on the SQL side keeps
  the network payload bounded. No concern.

## Verdict

**merge-as-is**

Correct, well-structured timezone fix. Author chose the right
trade-off (accept ~24× more rows on the wire to gain
timezone-correct bucketing client-side). Tests aren't shown in the
diff — there don't appear to be any unit tests for this graph
component in the repo, so this PR matches existing conventions.
A maintainer might suggest a small unit test for `formatDateKey`
+ `formatDateLabel` round-trip as a follow-up, but that's polish.
