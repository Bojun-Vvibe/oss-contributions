# Review: QwenLM/qwen-code PR #3852

- **Title:** fix(core): activate skills from discovered result paths
- **Author:** qiuqiuwen25
- **Head SHA:** `8a5fa3b1920ea25f5703e981641ee562c6c29d49`
- **Files touched:** 9
- **Lines:** +496 / -45

## Summary

Extends the skill-activation pipeline in
`packages/core/src/core/coreToolScheduler.ts` so that, after a tool
call completes, the scheduler considers not only the *input arguments*
of the tool call (e.g. the `file_path` for `read_file`, the `pattern`
for `glob`) but also any concrete file paths the tool actually
returned via `ToolResult.resultFilePaths`. Those discovered paths are
merged with the input paths, deduplicated, and passed to
`SkillManager.matchAndActivateByPaths(...)`.

The motivating case (visible in the new tests at
`coreToolScheduler.test.ts:4701-4768`) is `glob` returning files that
trigger a skill the input pattern alone would not have matched.

## Notes

- New tests cover the right shape:
  1. `'includes concrete result paths in skill activation candidates'`
     — verifies `matchAndActivateByPaths` is called with input
     pattern + each `resultFilePaths` entry.
  2. `'deduplicates overlapping input and result paths before
     activation'` — single call when input and result overlap.
  3. `'does not unescape concrete result paths before activation'`
     — preserves raw paths (e.g. `/proj/src/foo\\ bar.ts`) so that
     glob-escaping logic on the input side does not corrupt
     filesystem-literal result paths.
- The "no unescape" test is the most important: it locks the
  asymmetry between user-provided patterns (which may need glob
  unescaping) and tool-returned absolute paths (which must not).
- Public API surface change: `ToolResult` gains an optional
  `resultFilePaths?: string[]`. Adopting tools must opt in, so
  existing tools that don't set it are unaffected — backward
  compatible.
- Potential follow-up: confirm `matchAndActivateByPaths` is
  resilient when `resultFilePaths` contains thousands of entries
  (large glob results) — perhaps cap or stream.

## Verdict

**merge-after-nits** — solid feature with proper test coverage.
Worth a maintainer note about the upper bound on
`resultFilePaths` cardinality before merging.
