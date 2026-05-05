# sst/opencode PR #25924 — chore: add generation completion sentinels

- URL: https://github.com/anomalyco/opencode/pull/25924
- Head SHA: `2a1bc29bcf2cef4672a8b17f43ff670b8b65a12e`
- Size: +5 / -0

## Summary

Adds four `console.error(...)` sentinel lines to `script/generate.ts` and `packages/sdk/js/script/build.ts` so that downstream automation (CI logs, IDE-watch wrappers) can grep for known phase-complete markers instead of inferring success from process exit alone.

## Specific findings

- `packages/sdk/js/script/build.ts:55` — single `console.error("opencode sdk js build complete")` appended after the final `await $\`rm openapi.json\``. Correctly uses stderr (won't pollute stdout consumers that capture generated artifacts).
- `script/generate.ts:6` — `"opencode generate: sdk build complete"` after sdk build.
- `script/generate.ts:9` — `"opencode generate: openapi export complete"` after `bun dev generate > ../sdk/openapi.json`.
- `script/generate.ts:12-13` — `"opencode generate: format complete"` then `"opencode generate complete"` as final sentinels.

## Notes

- All four messages use the `opencode generate:` / `opencode sdk js build complete` prefix style — consistent and greppable.
- Choice of `console.error` over `console.log` is correct: stdout from `bun dev generate` is being redirected into `../sdk/openapi.json` at line 8, so any incidental stdout writes would corrupt the JSON file. Sentinels go to stderr and stay out of the artifact.
- Strings are not localized and not user-facing, so no i18n concern.
- No tests required for purely log-emitting changes.

## Verdict

`merge-as-is`
