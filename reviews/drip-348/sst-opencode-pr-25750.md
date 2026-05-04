# sst/opencode#25750 — chore: Update fff to 0.7.0 and removed runtime sorting

- **Head SHA**: `3a2796853013`
- **Verdict**: `request-changes`

## Summary

Bumps `@ff-labs/fff-bun` from 0.6.4 to 0.7.0 (`packages/opencode/package.json:110`), strips the JS-side dedup/glob/sort helpers in `packages/opencode/src/file/search.ts` now that the new picker handles them natively, and replaces the blocking `pick.waitForScan(5_000)` with a polling loop on a new `isScanning()` API. Also adds a `bench-search.ts` script (+96 lines) for measuring the new path.

## Findings

- `packages/opencode/src/file/search.ts:202-216` — `fffGlobbedQuery` is broken. It computes `resolvedGlob` from the array case and then ignores it in the return: `return \`${glob} ${query}\`` interpolates the original `glob` (which can be `string | string[]`). For an array glob, `${glob}` joins with `","`, not `" "`, so the picker receives e.g. `"**/*.ts,**/*.tsx hello"`. Should be `return \`${resolvedGlob} ${query}\``. This is the only consumer of the new helper (`:275`), so the bug ships with the PR.
- `packages/opencode/src/file/search.ts:235-243` — busy-poll on `isScanning()` every 25ms is fine when the scan finishes quickly, but `Effect.sleep("25 millis")` in a tight loop with a 5s budget is up to 200 wakeups for a slow project. The previous `pick.waitForScan(5_000)` returned a single Result. If the new native call is non-blocking by design, accept it; otherwise consider exponential backoff (25→50→100→…) and document why polling beat the blocking variant.
- `packages/opencode/test/file/index.test.ts:826` — `await sleep(100); // fff guarantees search corretness within 50ms after file write`. Typo (`corretness`) and the comment contradicts the magic number — if the SLA is 50ms, why 100? Either tighten to 50 or document the 2× safety margin.
- `packages/opencode/test/server/httpapi-file.test.ts:68` — `console.log(files);` left in a committed test. Remove before merge.
- `packages/opencode/src/file/search.ts:301` — formatter collapsed the multi-line `remember(...)` call into one line. Cosmetic, but unrelated to the PR's stated goal; worth noting in the description so reviewers don't chase it.
- `packages/opencode/src/file/index.ts:636` — switching `cwd: Instance.directory` → `cwd: (yield* InstanceState.context).directory` is a behavior change tucked into a "remove sorting" PR. Looks correct (matches the rest of the file) but deserves a one-line note in the PR body.

## Recommendation

The `fffGlobbedQuery` bug at `:212` will silently break grep-with-glob for any caller passing an array, which is exactly what the PR's removal of the JS-side `allow()` filter is trying to delegate to fff. Fix that, drop the stray `console.log`, and fix the test typo. The 0.7.0 bump itself looks fine and the dedup removal is justified by the new picker's semantics.
