# anomalyco/opencode PR #24273 — docs: correct compaction prune default

- **URL:** https://github.com/anomalyco/opencode/pull/24273
- **Head SHA:** `ee1c397e7be0db81d2dfe59af3b10054531187bd`
- **Files touched:** 21 (1 schema source, 2 generated SDK files, 18 i18n docs)
- **Verdict:** `merge-as-is`

## Summary

Doc-only fix: `compaction.prune` is disabled by default in code, but the
config schema description and all 18 localized doc pages claimed it
defaulted to `true`. PR realigns description text and regenerates SDK +
OpenAPI artefacts.

## Specific references

- `packages/opencode/src/config/config.ts:223` — single-source description
  flips from `(default: true)` → `(default: false)`. This is the canonical
  source the codegen reads from.
- `packages/sdk/js/src/v2/gen/types.gen.ts:1655` and
  `packages/sdk/openapi.json` — regenerated from the schema, identical
  string change. Confirms the codegen pipeline is wired correctly.
- 18 `packages/web/src/content/docs/<locale>/config.mdx` files — all flip
  the same line. Author kept en + i18n in lockstep, which avoids the
  common "translations drift" trap.

## Reasoning

Pure docs/codegen, zero behavioural risk. The default is set elsewhere in
runtime config and is not changed by this PR. Author confirmed `bun
typecheck` + `bun turbo typecheck` pre-push, which is sufficient given
the scope.

One micro-nit only worth mentioning if other reviewers raise it: i18n
files are mass-edited with the English string ("disabled by default")
instead of localized phrasing. That is consistent with how prior
compaction docs were written, so not a blocker.

Merge as-is.
