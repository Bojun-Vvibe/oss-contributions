# sst/opencode #25346 — chore: remove unused util/scrap.ts dummy file

- URL: https://github.com/sst/opencode/pull/25346
- Head SHA: `63beafe6ebfe49f48ac206698c13fd2e5aea73ec`
- Files: `packages/opencode/src/util/scrap.ts` (+0/-10)

## Context

`util/scrap.ts` exported four no-op placeholders (`foo: "42"`, `bar: 123`, `dummyFunction()` that `console.log`s, `randomHelper()` returning `Math.random() > 0.5`). Author confirms `grep -rn "util/scrap\|dummyFunction\|randomHelper" packages` returned nothing outside the file itself. The unrelated debug-only `cli/cmd/debug/scrap.ts` (the `ScrapCommand`) lives in a different namespace and is untouched — easy to confuse the two if you're skimming, but the rename collision is clean here.

## Risks

- None functional. Net `-10` and no public API surface (file isn't re-exported anywhere). Tree-shaken bundlers were already eliding this; the win is that human readers stop wondering whether `randomHelper()` is load-bearing somewhere.
- One residual nit worth callout in PR body: also confirm `tsconfig.json` `include` / any explicit-file `references` don't list `util/scrap.ts` so `tsc` doesn't error after deletion. The author ran `bun run typecheck` clean so this is empirically fine, but stating it locks the contract.

## Verdict

`merge-as-is` — clean dead-code excision, evidenced by negative grep and a clean typecheck. No alternative interpretation: the file is debug debris from someone's scratchpad checked in by accident, and removing it costs nothing.
