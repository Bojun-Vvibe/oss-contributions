# sst/opencode #24296 — docs: sync env vars with source code

- **PR:** https://github.com/sst/opencode/pull/24296
- **Head SHA:** `28cea66911be8047a9890cf17084a74b2cb7db18`
- **Files changed:** ~40 (`packages/web/src/content/docs/<lang>/cli.mdx` + `providers.mdx` across all locales) — `+1195/−830`

## Summary

Closes #24235. Reconciles the env-var tables in the docs site against `packages/opencode/src/flag/flag.ts` and `packages/opencode/src/provider/provider.ts`. Adds 8 missing main vars (`OPENCODE_AUTH_CONTENT`, `OPENCODE_DISABLE_EXTERNAL_SKILLS`, `OPENCODE_DISABLE_PROJECT_CONFIG`, `OPENCODE_MODELS_PATH`, `EXA_API_KEY`, three `OTEL_*`) and 2 experimental vars; sorts both tables alphabetically; propagates the same edits to every `<lang>/cli.mdx` + `providers.mdx`.

## Line-level call-outs

- `packages/web/src/content/docs/ar/cli.mdx:558` (representative — same hunk recurs in every locale) — adds `OPENCODE_AUTH_CONTENT` row and re-sorts the entire table. The sort is **case-sensitive** (`OPENCODE_*` before `EXA_*` before `OTEL_*` would be alphabetical, but here `EXA_API_KEY` and `OTEL_*` are appended **after** the `OPENCODE_*` block in source order, not interleaved alphabetically). The PR description claims "sorted both tables alphabetically" but the actual output is "sorted within the `OPENCODE_*` namespace, then non-`OPENCODE_*` appended". Either commit to true alphabetical (which interleaves `EXA_API_KEY` between `OPENCODE_DISABLE_*` and `OPENCODE_ENABLE_*`) or rename the claim to "grouped by prefix, sorted within group".
- `packages/web/src/content/docs/cli.mdx:558+` (English source of truth) — same content as the locale files. **Risk:** the locale files were edited by hand to mirror the English source. Any future change to the English table now has to be mirrored across ~20 locale files manually; the docs team historically uses an automation pass for this. Worth a follow-up issue: "auto-generate env-var tables from `flag.ts` schema". For this PR, accept the manual sync; for the next one, please don't keep paying the 20× edit cost.
- `OPENCODE_DISABLE_EXTERNAL_SKILLS` row added — verify this string matches the literal env name read in `flag.ts`. The PR doesn't show the source-code line it's claiming parity with. Reviewers should grep `OPENCODE_DISABLE_EXTERNAL_SKILLS` in `packages/opencode/src/flag/flag.ts` to confirm this isn't a typo (`SKILL` vs `SKILLS` is the kind of error that survives a docs-only review).
- `OPENCODE_PERMISSION` description says "تهيئة أذونات JSON مُضمّنة" (Arabic) — none of the locale translations were updated in this PR for the **new** rows. The Arabic, Bosnian, Danish, German, Spanish, etc. translations of the new entries are likely machine-translated drafts. Check in particular the `OTEL_*` rows: telemetry terminology is brittle in translation and a wrong description here will be copied into user configs.
- Non-trivial diff cost: `+1195/−830` for what is functionally a "+10 rows × N locales" change. The volume is genuine (alphabetical re-sort touches every line), not bloat — but it does mean a future bisect across this commit will be painful. Splitting into "alphabetize" + "add missing rows" would have made this much easier to review and revert in pieces.

## Verdict

**merge-after-nits**

## Rationale

This is documentation hygiene work that genuinely closes a parity gap, and the volume is justified by the alphabetical re-sort. Two nits before merge: (1) clarify whether the sort is "alphabetical within prefix" or "fully alphabetical" — the description claims the latter, the diff implements the former; (2) confirm the new env-var **names** match `flag.ts` literals (a single typo here ships to 20 languages). The translation quality of the new descriptions is a real concern but is outside the scope of a docs-sync PR; file a follow-up rather than block on it. With the sort claim fixed and the names spot-checked, this should land.
