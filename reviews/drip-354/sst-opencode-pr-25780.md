# sst/opencode PR #25780 — fix(i18n): correct Japanese translation for todo progress

- Repo: `sst/opencode`
- PR: https://github.com/sst/opencode/pull/25780
- Head SHA: `c813072a3a6bd1d31129a4a3d622a35f49cc51c0`
- Size: +1 / -1 (single line)

## Summary

One-line fix in `packages/app/src/i18n/ja.ts:790`. The
interpolation order for `session.todo.progress` is swapped from
`{{done}} 個中 {{total}} 個の Todo が完了` to
`{{total}} 個中 {{done}} 個の Todo が完了`.

## Analysis

The original string parses literally as "Out of {done}, {total}
Todos are complete" — which inverts the meaning when, e.g., 2/5
Todos are done (it would render "2個中 5個の Todo が完了" =
"out of 2, 5 Todos are complete", which is nonsense). The fix
restores the natural Japanese phrasing "{total}個中 {done}個" =
"{done} out of {total}".

Cross-checked the same key in adjacent locales for parity:

- `session.question.progress` on the next line uses
  `"{{total}} 問中 {{current}} 問"` — same `total中 current`
  pattern, so this fix aligns with the existing convention.
- The English source (would be in `en.ts`) almost certainly reads
  `"{{done}} of {{total}} todos completed"`, which English speakers
  parse as "{done}/{total}" — but Japanese particle 中 ("out of")
  reverses operand order, so a literal placeholder-order copy is
  the bug.

## Risk / surface

- Pure data change in a translation table — no code paths affected.
- Both placeholders (`done`, `total`) still exist and are still
  unique in the string, so any i18n-key linter that validates
  placeholder coverage will pass.
- No test added, but a rendering snapshot for one locale string
  would be overkill.

## Verdict

**merge-as-is** — trivial, correct, follows the existing
`question.progress` precedent on the very next line.
