# Review: sst/opencode #25581 — fix(vcs): avoid unbounded diff memory usage

- **Repo**: sst/opencode
- **PR**: #25581
- **Head SHA**: `c6c09f88ab33042982e01062a06fba1e2745b1ec`
- **Author**: nexxeln

## What it does

Replaces the previous `vcs.ts` path that read every changed file from disk
(via `AppFileSystem.readFile`) and ran the JS `diff` library to fabricate
full-file patches. The new path delegates patch generation to native git
through three new `Git.Interface` methods (`patch`, `patchAll`,
`patchUntracked`) plus `statUntracked`, all of which honor a per-call
`maxOutputBytes` cap and surface a `truncated: boolean` on the result.

## Diff notes

- `packages/opencode/src/git/index.ts:113-145` — rewires `run()` to fold
  stdout/stderr through a byte-counted accumulator. When
  `opts.maxOutputBytes` is set, the fold keeps appending byte counts but
  only retains slices up to the cap, then flips `truncated`. Clean and
  back-compat: callers without `maxOutputBytes` get the old behavior.
- `git/index.ts:273-310` — `patch` and `patchUntracked` deliberately
  return `text: ""` when `truncated`, while `patchAll` keeps the
  partial text. That asymmetry is intentional (per-file consumers can't
  meaningfully use a half-patch; the whole-tree consumer uses the
  partial as a best-effort summary), but it is undocumented in the
  type. A one-line comment near the `Patch` type at `git/index.ts:49-52`
  would prevent a future reader from "fixing" it.
- `vcs.ts:13-15` — `PATCH_CONTEXT_LINES = 2_147_483_647` (i.e.
  `INT_MAX`) is passed to git's `--unified=` flag to emulate the old
  full-file-context behavior without actually reading the files. That's
  the right trade, but the constant is a foot-gun magic number; consider
  `Number.MAX_SAFE_INTEGER` or a named export from `Git` so it's
  obvious why the value is what it is.
- The `quoted`/`unquote`/`diffFile`/`chunkFile` helpers
  (`vcs.ts:34-90` area) are a hand-rolled mini parser for git's
  `core.quotePath` output. The escape table covers `\t \n \r " \\` —
  missing `\a \b \f \v` and octal escapes (`\NNN`). Real-world impact
  is small (filenames rarely contain those), but it would be safer to
  unconditionally `git -c core.quotePath=false` on the diff invocations
  and avoid parsing escapes at all. Worth a follow-up.

## Concerns

1. The two byte caps (`MAX_PATCH_BYTES`, `MAX_TOTAL_PATCH_BYTES`) are
   both 10 MB and not configurable. Reasonable defaults, but a power
   user with a giant monorepo refactor will silently lose patches with
   no way to opt out. Either expose via `OPENCODE_VCS_MAX_PATCH_BYTES`
   env or document the cap in the diff response.
2. `truncated: false` is added to the `fail()` Result at
   `git/index.ts:24-30`. Correct (a failed spawn produced no output to
   truncate), but consumers who key off `truncated` to decide "render
   anyway" will need to also check `exitCode !== 0`. Not addressed in
   this PR — assumption seems fine for now since callers go through
   `Effect.fn` and surface failures upstream.
3. Test coverage exists per the PR description (`bun test
   test/git/git.test.ts`, `bun test test/project/vcs.test.ts`), but
   the diff doesn't include the test changes — confirm the tests
   actually exercise the truncation path (e.g. inject a small
   `maxOutputBytes` and assert `truncated === true`). If they don't,
   ask for one before merge.

## Verdict

merge-after-nits
