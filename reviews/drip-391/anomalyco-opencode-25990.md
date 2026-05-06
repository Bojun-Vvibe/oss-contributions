# anomalyco/opencode#25990 — chore: fix model alerts

- URL: https://github.com/anomalyco/opencode/pull/25990
- PR: #25990
- Author: vimtor (Victor Navarro)
- Head SHA: `4626757f301fdaf08aa8331865329ed8425daada`
- State: OPEN  | +7 / -11

## Summary

Two-commit infra change to Honeycomb model-error alerting and the Discord incident webhook:

1. `infra/monitoring.ts:215-300` — flips Honeycomb trigger semantics from `on_change` (50% percentage delta vs a baseline window of 3600 or 86400 seconds) to `on_true` with a static absolute threshold of `0.8`. Drops the `baseline: 3600 | 86400` field from the `Trigger` type, drops `baselineDetails`, and tightens `frequency` from `900` → `300`.
2. `packages/console/app/src/routes/incident/webhook.ts:5,42` — replaces `@everyone` with a specific role mention `<@&1501447160175136838>` (named `DISCORD_INCIDENT_ROLE_ID`), reducing notification blast radius.

## Specific references

- `monitoring.ts:218`: removes `baseline: 3600 | 86400` from `Trigger`.
- `monitoring.ts:252-254`: `threshold: { op: ">=", value: 50 }, baseline: 3600` → `threshold: { op: ">=", value: 0.8 }`.
- `monitoring.ts:295-298`: `alertType: "on_change"` + `frequency: 900` + `baselineDetails: [...]` → `alertType: "on_true"` + `frequency: 300`. No `baselineDetails`.
- `webhook.ts:5`: new module-level `const DISCORD_INCIDENT_ROLE_ID = "1501447160175136838"`.
- `webhook.ts:42`: `"@everyone"` → `` `<@&${DISCORD_INCIDENT_ROLE_ID}>` ``.

## Concerns / nits

- **Threshold unit change is the load-bearing semantic**: the `httpErrors` formula is `ERROR = $FAILED / $TOTAL` (a ratio), so `0.8` means "alert when 80% of requests in the 900s window fail". The previous `value: 50` was correct only under `on_change` percentage-delta semantics. The new value is internally consistent with `on_true` on a ratio — confirm in the PR body that this is intentional and that `0.8` is the desired floor (vs e.g. `0.05` for a 5% error rate). 80% fail-rate is *very* permissive; a real outage at 50% fail will not page until it gets worse.
- `frequency: 300` (5 min) is 3× more frequent than before — fine for `on_true` but worth confirming the alerting destination (PagerDuty, Discord) tolerates the cadence for noisy partial outages.
- Hardcoded Discord role ID is fine for an infra file but consider moving to `Resource.*` env to mirror how other Discord IDs are sourced (the file already imports `Resource`).
- No CHANGELOG / changeset entry; chore-scoped infra PR so probably fine.

## Verdict

**merge-after-nits (man)** — clean refactor; the `0.8` threshold value should be confirmed (likely meant to be lower) before this lands.
