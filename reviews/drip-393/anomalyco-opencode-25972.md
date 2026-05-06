# Review: anomalyco/opencode#25972

- **PR:** [fix(desktop): suppress browser API Sentry errors in prod](https://github.com/anomalyco/opencode/pull/25972)
- **Head SHA:** `e969d0af604c1278449b83622a84e0b64a6a0b31`
- **Merged:** 2026-05-06T04:44:40Z
- **Verdict:** `merge-after-nits`

## Summary

Extends the existing prod-only Sentry integration filter to also drop `BrowserApiErrors` alongside `GlobalHandlers`. Keeps non-prod behavior identical so devs still see the noisy reports.

## Specific notes

- `packages/desktop/src/renderer/index.tsx:43-50` — single conditional, restructured to `i.name !== "Breadcrumbs" && !(prod && (GlobalHandlers || BrowserApiErrors))`. Logic is correct: in prod, both `GlobalHandlers` and `BrowserApiErrors` are filtered; everywhere else only `Breadcrumbs` is dropped.
- The two-name OR is fine for now but if a third integration name ever joins this list, a `Set` lookup would scale better. Tiny nit, not blocking.
- No test coverage, but the renderer Sentry init is genuinely hard to unit-test without mocking the SDK's integration registry. Acceptable.

## Rationale

Right call: `BrowserApiErrors` is the integration that wraps `setTimeout`/`requestAnimationFrame`/etc. and reports their throws. In an Electron renderer those callbacks routinely fail during teardown for reasons that are not actionable (window closing, contexts going away). Filtering it in prod cuts noise without hiding actual handled errors that the SDK would still capture via `captureException`.

Nit: a brief comment naming *why* `BrowserApiErrors` is being dropped (so the next person doesn't re-add it during a refactor) would be valuable. Worth a follow-up but not a blocker.
