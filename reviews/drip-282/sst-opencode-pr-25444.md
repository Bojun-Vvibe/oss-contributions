# sst/opencode PR #25444 — Use instance test helper in grep tests

- Head SHA: `02914cc638ceded6245649ba6b6926b07467b84a`
- Size: +50 / -53, 1 file (`packages/opencode/test/tool/grep.test.ts`)

## Specific refs

- `test/tool/grep.test.ts:5` — import swap: drop `provideTmpdirInstance`, keep `provideInstance` (still used by the non-tmpdir tests in the same file), add `TestInstance`.
- `test/tool/grep.test.ts:57-73` ("no matches returns correct output") — `it.live` → `it.instance`; `dir` lifted to `test.directory`.
- `test/tool/grep.test.ts:76-92` ("finds matches in tmp instance") — same conversion.
- `test/tool/grep.test.ts:95-111` ("supports exact file paths") — same conversion; preserves the `Line 2: line2` substring assertion.

## Assessment

Mirror of PR #25445 for the grep test file. `provideInstance` is intentionally retained on the first test in the same describe (not in the diff window but visible in the diff context) — only the tmpdir-flavored variants get migrated. Clean port, no behavior change.

Same nit as #25445 about cleanup semantics being implicit in `TestInstance`.

verdict: merge-as-is