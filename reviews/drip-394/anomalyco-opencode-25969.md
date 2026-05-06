# Review: anomalyco/opencode #25969 — go: restore Kimi K2.6 limits

- Head SHA: `e062fff31dd9f39010d738eafd523d2dbbd9a004`
- Files: `packages/console/app/src/i18n/{ar,br,da,de,en,es,fr,it,ja,ko,no,pl,ru,th,tr,zh,...}.ts`

## Summary

Bulk i18n cleanup that removes the now-stale "Kimi K2.6: 3× usage limit
through April 27" promotional banner copy from every locale dictionary. The
commit deletes exactly two adjacent string keys per locale file
(`go.banner.badge` and `go.banner.text`) at the same insertion point in each
file (right after `go.cta.promo`/`go.pricing.body`, immediately before
`go.graph.free`).

## Specifics

- `packages/console/app/src/i18n/en.ts:60-62` — drops `"go.banner.badge"`
  ("3x") and `"go.banner.text"` ("Kimi K2.6 gets 3× usage limits through
  April 27"). The deletion position matches every other locale exactly, which
  is the right shape for an i18n key-removal sweep (no per-locale insertion
  drift).
- `packages/console/app/src/i18n/ar.ts:264-266`, `br.ts:268-270`,
  `da.ts:266-268`, `de.ts:268-270`, `es.ts:269-271`, `fr.ts:270-272`,
  `it.ts:266-268`, `ja.ts:265-267`, `ko.ts:262-264`, `no.ts:266-268`,
  `pl.ts:267-269`, `ru.ts:270-272`, `th.ts:264-266`, `tr.ts:268-270`,
  `zh.ts:255-257` (truncated diff, same shape) — uniform `-2 lines` shape;
  no whitespace drift, no stray quote re-encoding visible in the visible
  window.

## Nits

- The PR title says "restore Kimi K2.6 limits" but the actual change is
  removing the *banner copy* announcing the temporary 3× limit (presumably
  because the limit period ended on April 27). This is purely a presentation
  cleanup; if there is a sibling change that flips the actual
  per-model rate-limit knob in the backend, that should be cross-linked in
  the PR body so a future reviewer doesn't think the *limit itself* was
  reverted in this PR.
- No corresponding source-code grep guard. If a `t("go.banner.badge")` call
  remained in `*.tsx`, this PR would silently ship as a missing-translation
  fallback at runtime in every non-English locale. Worth a `rg "go.banner"
  packages/console` in CI on i18n-key-deletion PRs.
- Locales not visible in the truncated 200-line window
  (`pt`, `vi`, `zh-tw`, `id`, etc., if present) — confirm parity by
  diff line count: should be `(N_locales) × 2` removed lines total.

## Verdict

`merge-after-nits`
