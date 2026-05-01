# sst/opencode#25260 — fix: correct documentation typos

- **PR**: https://github.com/sst/opencode/pull/25260
- **Head SHA**: `d6e4252e74418be4019528ed8b6014d67159044d`
- **Size**: +4 / -4, 2 files
- **Verdict**: **merge-as-is**

## Context

Pure documentation typo cleanup, closes #25259. Two files touched:
- `packages/opencode/src/sync/README.md`
- `packages/web/src/content/docs/providers.mdx`

## What's right

Five edits, all surgical English-spelling corrections with zero behavior surface:

1. `sync/README.md:97` — `throught the bus` → `through the bus`. Docs in the sync subsystem describing event re-publication, no code path touches this string.
2. `sync/README.md:115` — `contain to full object` → `contain the full object`. The grammatical fix in the back-compat note about `session.updated` event reshaping in `server/projectors.js`.
3. `sync/README.md:117` — `even only contains` → `event only contains`. Same paragraph, missing `t`.
4. `sync/README.md:117` — `defintiion` → `definition`. Loader reference for zod schemas. Single token, no callers.
5. `providers.mdx:1770` — `soverign hosting` → `sovereign hosting` in the STACKIT provider section. Customer-visible docs site, the typo was sitting next to the correctly-spelled `data sovereignty` two phrases later in the same sentence — clear authoring slip.

The PR description's claim "intentionally limited to typo/documentation fixes and avoids unrelated cleanup" is borne out by the diff: net +4/-4, no whitespace churn, no reflow, no MDX code-fence touches.

## Risks / nits

- None. `bun turbo typecheck` was reported green by the author per the PR description, and the diff confirms only `.md`/`.mdx` text content changed — no import statements, no MDX components, no front-matter.
- Five typos for a docs PR is a comfortable batch size; the alternative (one PR per typo) would be noisier for reviewers.

## Verdict

**merge-as-is.** Documentation-only, mechanical, fully scoped. Safe to land without a follow-up.
