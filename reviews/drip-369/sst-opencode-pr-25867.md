# Review: sst/opencode PR #25867

- **Title:** fix(git): replace mutating Stream.runFold with Stream.runForEach
- **Author:** stephanschielke
- **Head SHA:** `1e1dca64f2ccd954fd943eff65f2f34e280fe18c`
- **Files touched:** 1
- **Lines:** +19 / -17

## Summary

In `packages/opencode/src/git/index.ts:114-138` the `collect` helper
used `Stream.runFold` with an accumulator that was *mutated in place*
(`acc.chunks.push(...)`, `acc.bytes += ...`). Effect's `runFold`
contract treats the accumulator as immutable; mutating the prior value
breaks structural-sharing assumptions and can produce inconsistent
results under fiber interleaving. The PR rewrites `collect` as an
`Effect.gen` that wraps `Stream.runForEach` over locally-scoped
`chunks`/`bytes`/`truncated` variables and returns the final
`{ buffer, truncated }`.

## Notes

- Semantics preserved: `maxOutputBytes === undefined` path still
  appends every chunk; bounded path still slices the remaining window
  and flips `truncated` once the cumulative byte count crosses the
  cap.
- Closure variables are scoped per-`collect` invocation, so the two
  parallel `collect(handle.stdout)` / `collect(handle.stderr)` calls
  in the `Effect.all` at line 137 do not share state.
- Suggest adding a regression test that exercises a single chunk
  larger than `maxOutputBytes` to lock the slice + truncated-flag
  behavior, but that is a follow-up.

## Verdict

**merge-as-is**
