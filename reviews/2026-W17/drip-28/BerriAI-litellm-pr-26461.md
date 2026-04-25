# PR #26461 — fix(ci): support CircleCI rerun failed tests for local_testing jobs

- Repo: BerriAI/litellm
- Head SHA: 68d4420233559b67b3a17b79041b764654863764
- Files touched: 1 (+3 / -3)

## Summary

CircleCI's "Rerun failed tests" feature has been broken for
`local_testing_part1`, `local_testing_part2`, and
`litellm_router_testing` — reruns collect 0 items, exit code 123.
Root cause: the `circleci-tests-plugin-cli` rerun pipes JUnit
`classname` dot-notation (`tests.local_testing.test_router`) into
`xargs pytest`, which expects file paths
(`tests/local_testing/test_router.py`). pytest collects nothing
and the job fails. Fix inserts a single `awk` filter between
`circleci tests run` stdout and `xargs pytest` in the three
affected job blocks of `.circleci/config.yml`.

## Specific findings

- `.circleci/config.yml:220`, `:301`, `:466` — all three sites
  swap:
  ```diff
  - --command="xargs uv run --no-sync python -m pytest \
  + --command="awk '/\\.py/ {print; next} {sub(/\\.[A-Z][^.]*$/, \"\"); gsub(/\\./, \"/\"); print $0 \".py\"}' | xargs uv run --no-sync python -m pytest \
  ```
  The awk preprocessor:
  1. Lines matching `/\.py/` (initial-run input — already file
     paths) pass through unchanged.
  2. Other lines (rerun input — dotted `classname`) get
     `[A-Z][^.]*$` (assumed test class suffix) stripped, then
     `.` → `/`, then `.py` appended.
- The PR body's verification snippet shows
  `tests/local_testing/test_router.py` and
  `tests.local_testing.test_router` both producing
  `tests/local_testing/test_router.py`. That's correct for the
  module case.
- **Actual diff differs from PR body verification**: the body
  shows a simpler awk
  (`awk '/\.py/ {print; next} {gsub(/\./, "/"); print $0 ".py"}'`)
  but the diff has an additional
  `sub(/\.[A-Z][^.]*$/, "")` step that strips a trailing
  CapitalizedClass component before the dot-to-slash conversion.
  That's the real production version (handles JUnit
  `classname="tests.local_testing.test_router.TestRouter"` →
  drop `.TestRouter`, then path-ify) but the body doesn't
  document it. Reviewer should ask the author to update the
  PR description with the regex that's actually shipping.

## Risks / nits

- The class-name strip uses `[A-Z][^.]*$`. That's fragile:
  1. Test files that start with capitals
     (e.g. `tests/local_testing/Test_router.py`) won't match
     the heuristic correctly because the basename itself
     starts with a capital — the regex strips the basename.
     Confirm none of the existing tests are named that way.
  2. PEP 8 class names that don't start with a capital (rare
     but possible — `def test_foo` style with no class
     wrapper produces classnames like
     `tests.local_testing.test_router` which has no trailing
     `[A-Z][^.]*` component — those go through the strip as a
     no-op, which is what we want).
- All three jobs get the identical filter. If a fourth job
  added later misses the change, it'll silently break reruns
  again. Worth extracting into a CircleCI command/parameter
  block, but that's a follow-up.
- No test added. CI-config-only changes don't typically get
  tests, but confirming the awk on a representative JUnit XML
  (incl. parametrized test names with `[`/`]`/`.` inside the
  parameters) before merge would prevent the next surprise.

## Verdict

**merge-after-nits** — The fix is in the right place and the
preprocess shape is right. Update the PR body to match the
actual awk that's shipping (the class-strip step), and verify
parametrized test names containing `.` inside `[...]`
parameters survive the `gsub`.

## What I learned

When a third-party tool emits identifiers in a syntax that
your downstream tool doesn't accept, an awk preprocessor in
the CI shim is the lowest-blast-radius fix — but the
correctness depends entirely on the structure of the *real*
identifiers, which include every weird parametrized-test name
your codebase generates. The verification snippet in the PR
body should always be the exact regex that ships, not a
simplified version, otherwise reviewers can't tell if the
edge cases were considered.
