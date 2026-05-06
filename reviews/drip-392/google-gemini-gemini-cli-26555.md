# google-gemini/gemini-cli #26555 — Actions Cost Reduction: CI Matrix Optimization

- PR: https://github.com/google-gemini/gemini-cli/pull/26555
- Head SHA: `6342ac20625c02a589b2c8e55eeeda2f902b9e53`
- Base: `main`
- Size: +8 / -22 across 1 file
- Files: `.github/workflows/ci.yml`

## Verdict
**needs-discussion**

## Rationale
This PR cuts macOS CI runners by collapsing a 3 Node × 2 shard matrix (6 jobs) down to a single Node 20.x job, justified as ~83% Mac-runner cost reduction. The Linux matrix is preserved unchanged. The mechanical edit is clean — the PR correctly removes the `strategy.matrix` block at `.github/workflows/ci.yml:243-251`, replaces all `${{ matrix.node-version }}` / `${{ matrix.shard }}` references with literal `20.x`, and folds the conditional sharded test invocation back into a single `npm run test:ci -- --coverage.enabled=false` + `npm run test:scripts` call (lines 264-265). Job names, artifact names, and the test reporter title are all updated consistently. No dangling matrix references.

But the *decision* to drop coverage on Node 22 and 24 for macOS deserves a maintainer conversation, not a quiet merge:

**1. This deletes the only macOS coverage of the support matrix the project ships against.**
Node 22 LTS and Node 24 are both currently supported runtimes per the `engines` field that other recent PRs in this same drip cycle (qwen-code's ink upgrade) explicitly bump *to* `>=22`. Linux-only coverage of Node 22/24 catches Linux-specific bugs only. The macOS-only bugs that have historically fired on Node 22+ (notably around `node:fs` on case-insensitive APFS, `child_process` PATH resolution differences, and the keychain integration that PR #26571 just fixed in this same drip cycle) would go undetected until a user reports them.

**2. "macos-latest-large" runner cost is real but the framing oversells the savings.**
The PR claims "approximately 83% reduction (from 6 jobs to 1)". That's a per-CI-run ratio, but PRs that don't touch macOS-relevant code already get fast `merge_queue_skipper` short-circuits, so the actual budget delta is smaller than the headline. Worth checking whether the saved budget is actually the bottleneck the project is hitting.

**3. There's a less destructive middle ground.**
Options worth weighing before this lands as-is:
- Keep Mac on Node 20 + Node `engines.minimum`, drop Node 24 only (Node 24 is current/non-LTS; until it becomes LTS, dropping it is defensible).
- Keep all 3 Node versions but drop the `cli` / `others` shard (so 3 jobs instead of 6 — half the savings, full coverage).
- Move Node 22/24 macOS jobs to scheduled (nightly) instead of per-PR, so PR latency cost goes to zero but regressions are still caught within 24h.
- Use `if: contains(github.event.pull_request.changed_files, '...')`-style targeting so the full Mac matrix only runs when macOS-touching files change.

**4. `continue-on-error: true` is preserved at line 243.**
This means even the single retained Mac job is non-blocking. Combined with reduced coverage, the practical effect on PR gating is: Mac regressions won't block merges and won't be detected on Node 22/24 at all. The PR doesn't acknowledge this.

**5. Drive-by claim that's hard to verify from this diff alone.**
The PR body says "Reverted unrelated changes to policy engines, documentation, and tests that were inadvertently included in previous iterations of this branch" and "optimizations in `gemini-cli-bot-pulse.yml` and `gemini-cli-bot-brain.yml` have been fully reverted to their baseline state". The visible file list is `.github/workflows/ci.yml` only, which is consistent with that claim, but it's worth a maintainer cross-check against the branch history to confirm nothing else snuck in.

## Specific lines
- `.github/workflows/ci.yml:233` — job display name collapses from templated to literal `Test (Mac) - 20.x`. Correct mechanical edit.
- `.github/workflows/ci.yml:243` — `strategy.matrix` block deleted entirely. The behaviour-changing line.
- `.github/workflows/ci.yml:253-254` — Node setup now hardcoded to `20.x`.
- `.github/workflows/ci.yml:263-265` — sharded test invocation collapsed into `npm run test:ci -- --coverage.enabled=false` + `npm run test:scripts`. Note this changes test execution from per-package targeted (`--workspace "@google/gemini-cli"` etc.) to whatever the root `test:ci` script does — verify root `package.json`'s `test:ci` actually covers all the workspaces the previous shard explicitly listed (`@google/gemini-cli`, `@google/gemini-cli-core`, `@google/gemini-cli-a2a-server`, `gemini-cli-vscode-ide-companion`, `@google/gemini-cli-test-utils`). If `test:ci` skips one of these, the Mac job is now silently undertesting.
- `.github/workflows/ci.yml:294, 304, 312` — artifact and reporter names updated. Correct.

## Suggested path forward
Have a maintainer weigh in on (a) which Node versions are actually mandatory for macOS PR gating vs nightly, (b) whether root `test:ci` exercises the same set of packages as the deleted shard's explicit `--workspace` list, and (c) whether dropping the matrix is preferable to keeping the matrix but dropping the shard. If those three questions get clean answers, this is a fast `merge-after-nits`.
