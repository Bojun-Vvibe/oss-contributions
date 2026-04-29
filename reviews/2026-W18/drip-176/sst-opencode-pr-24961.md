# sst/opencode#24961 — feat(tui): show provider quota in prompt metrics

- PR: https://github.com/sst/opencode/pull/24961
- Head SHA: `34d9340d3e39a0699c0f7b6d868032e5c894f4f5`
- Author: 50sotero
- Base: `dev`
- Size: +1278 / -65 across 15 files

## Summary

Adds a compact provider-quota strip to the TUI prompt-metrics row. Wires
`syncCodexQuota()` and `syncProviderQuota()` into the SDK sync context with a
60s `CODEX_QUOTA_REFRESH_INTERVAL`, exposes the data through new
`console_state.codexQuota` / `providerQuota` slices, and renders via the new
`metrics.ts` helpers (`formatProviderQuotaMetrics`,
`formatCodexQuotaMetrics`). Also threads two `Awaited<ReturnType<...>>` type
annotations into `packages/console/resource/resource.node.ts:37` and `:59` to
keep root typecheck happy after a `client.kv` API surface change.

## Notable design choices

- `metrics.ts:53-57` — the `codexQuotaLayout(width)` ladder (`>=200` → 10-col
  bar + timestamp; `>=120` → 5-col + timestamp; `>=90` → timestamp only;
  else just `5h N% · wk N%`) is a clean way to make the strip survive narrow
  terminals without conditional CSS-style hackery. Worth noting that the
  function is reused for *provider* layouts at `metrics.ts:69` even though
  it's named `codexQuotaLayout` — minor naming smell.
- `sync.tsx:120-131` — `codexQuotaWorkspace` / `providerQuotaWorkspace` act
  as a per-workspace re-entrancy guard: the function early-returns if
  `current === codexQuotaWorkspace` and only releases in the `finally`. This
  prevents stampedes when the user rapidly switches workspaces, but the
  semantics are subtle ("we're already syncing for this workspace, skip").
- `metrics.ts:88-94` — only `confidence: "exact" | "reported"` quota windows
  surface in the prompt strip. `"estimated"` is filtered out, which is the
  right call (don't show users a number they shouldn't trust).

## Concerns

1. **`metrics.ts:38` `Math.round` direction.** `formatQuotaBar` uses
   `Math.round((percent/100) * columns)`, which means a user at 99.4% of
   quota with a 10-column bar shows 10/10 filled (i.e., looks full). Probably
   want `Math.floor` here so "100% bar = exhausted" remains a one-shot
   signal.
2. **Silent swallow in `syncProviderQuota` `catch {}` at `sync.tsx:139`.**
   Compare to `syncCodexQuota` which has *no* catch — codex errors will
   propagate, provider errors won't. Pick one policy. If the intent is
   "provider quota is best-effort, don't crash the TUI," log via
   `Log.create("sync.providerQuota").error(err)` so it's at least debuggable.
3. **`sync.tsx:114-115` `let codexQuotaWorkspace: string | null | undefined`
   tri-state.** Using `undefined` to mean "no sync in flight" and `null` to
   mean "no workspace selected" is clever but easy to break in a future
   refactor. A typed enum or a `Map<string, Promise<void>>` of in-flight
   syncs would be more obvious.
4. **No `CODEX_QUOTA_REFRESH_INTERVAL` consumer in the diff hunk shown.**
   The constant is declared at `sync.tsx:36` but the actual `setInterval` /
   `setTimeout` wiring isn't visible in the first 250 lines of the diff —
   confirm it exists and that `onCleanup` (newly imported at line 30)
   actually clears the timer; otherwise stale workspaces will keep polling.
5. **15-file footprint for a "compact metric row".** The SDK regen
   (`sdk.gen.ts +62`, `types.gen.ts +77`, `openapi.json +189`) inflates the
   diff but is unavoidable given the `experimental.providerQuota` route is
   new. Worth calling out in the PR body that reviewers should focus on the
   four hand-written files (`metrics.ts`, `quota.ts`, `experimental.ts`,
   `sync.tsx`).
6. **Active-provider matching at `metrics.ts:74-78`.** Lowercase compare
   on both `provider` *and* `label` is too permissive — if a user has two
   providers whose labels collide after lowercasing (e.g., `OpenAI` and
   `openai`), the wrong one wins. Match on `provider` (the stable ID) only
   and treat `label` as display-only.

## Verdict

**merge-after-nits** — solid feature with good test coverage (35 pass / 0
fail across 5 test files spanning sync, metrics formatting, plugin, server,
and SDK). The bar-rounding, silent-catch, and timer-cleanup items above are
small but worth fixing before merge so this doesn't land a regression mode
on day 1. Naming nit (`codexQuotaLayout` used for provider layout) can be
deferred.
