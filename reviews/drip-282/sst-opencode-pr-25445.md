# sst/opencode PR #25445 — Use instance test helper in glob tests

- Head SHA: `21e87745f0cbc8ecf47c2b5552652f3fc1b4a26a`
- Size: +38 / -40, 1 file (`packages/opencode/test/tool/glob.test.ts`)

## Specific refs

- `test/tool/glob.test.ts:11` — swap `provideTmpdirInstance` import for `TestInstance`.
- `test/tool/glob.test.ts:36-52` ("matches files from a directory path") — converted from `it.live` + `provideTmpdirInstance(dir => …)` callback wrapper to `it.instance` + `yield* TestInstance` access. `dir` becomes `test.directory`.
- `test/tool/glob.test.ts:55-79` ("rejects exact file paths") — same pattern conversion; `Exit.isFailure` assertion path preserved.
- Net change is `-40 / +38`, all in one file. No prod code touched.

## Assessment

Mechanical port to the new `it.instance(...)` helper, identical behavior. Symmetric with PR #25444 which does the same to `grep.test.ts`. Author ran `bun typecheck` and the file's tests; pre-push hook ran `bun turbo typecheck`. Low risk — pure test-infra refactor.

Nit: the conversion drops the implicit "tmpdir is created and cleaned up by `provideTmpdirInstance`" guarantee in favor of whatever `TestInstance` provides. Worth confirming `TestInstance` also handles cleanup (it presumably does given #25444 lands the same way), but a one-line comment in `fixture.ts` would help future readers.

verdict: merge-as-is