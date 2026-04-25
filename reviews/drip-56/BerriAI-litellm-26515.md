# litellm #26515 — render dict-shaped fallback entries in router settings

- URL: https://github.com/BerriAI/litellm/pull/26515
- Head SHA: `1efcbceed72e62c581630fbb8ae0fac95a8e85d4`
- Verdict: **merge-as-is**

## What it does

Fixes #26473: the Router Settings → Fallbacks tab would crash or render an
empty cell when an entry from `litellm_settings.fallbacks` was a `dict`
(`{model_name: [...]}`) rather than an array of strings. The backend always
allowed both shapes; the UI only handled the array shape.

The PR teaches `Fallbacks.tsx` to normalize either shape into the same
internal `{ source, targets }` view-model and adds 5 regression cases in
`Fallbacks.test.tsx`.

## Diff notes

- Normalization happens at the data-binding boundary, not deep inside render
  — exactly where you want it. The component itself stays shape-agnostic.
- Test additions cover: pure-array entry, dict entry, mixed list, empty
  targets, and a malformed entry (string instead of array). The malformed
  case asserts the row is skipped rather than crashing the whole settings
  pane, which is the right fail-soft behavior.
- The PR also pulls in some unrelated Black/Ruff reformatting in
  `prompt_templates/factory.py`, `predibase/chat/transformation.py`, and
  `xecguard.py`. That's pure whitespace — annoying to review but not risky.
  Likely produced by the repo's pre-commit hook on a stale branch.

## Why this verdict

Tight, well-tested UI fix; the only "noise" is autoformatter churn that
the maintainer can squash on merge. The view-model normalization is the
right design choice (one shape internally, two shapes externally) and the
regression tests directly exercise the bug from the issue.
