# Review: sst/opencode PR #25869

- **Title:** docs: fix opencode README
- **Author:** andres-cq
- **Head SHA:** `82caff4c9a2bbd241d1f43451b4b0496370ab3ca`
- **Files touched:** 1
- **Lines:** +1 / -1

## Summary

One-line fix in `packages/opencode/README.md:12` changing the
quickstart command from `bun run index.ts` to `bun run src/index.ts`,
which matches the actual entry-point location in the package. No
behavior change; trivial docs correction.

## Notes

- Path is correct: `packages/opencode/src/index.ts` exists and is the
  bun entry point referenced from `package.json`.
- No risk of regression — single doc line.
- No missing-trailing-newline issue introduced.

## Verdict

**merge-as-is**
