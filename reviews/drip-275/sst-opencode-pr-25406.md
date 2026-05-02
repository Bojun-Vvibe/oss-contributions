# Review: sst/opencode #25406

- **PR:** sst/opencode#25406 — `fix(beta): resolve missing ${tmp} in shell prompt template`
- **Head SHA:** `7aab16d17485dfb86415fcee9d29bf9764c3ff68`
- **Files:** ~100 (+4501/-1446)
- **Verdict:** `needs-discussion`

## Observations

1. **Title vs scope mismatch (significant)** — The stated fix is a one-line addition of `tmp: Global.Path.tmp` into `ShellPrompt.render()` values, plus a regression test. But the diff touches >100 files including a wholesale removal of `build-tauri` from `.github/workflows/publish.yml` (-179 lines), addition of `repo_clone.ts` (+207), `repo_overview.ts` (+238), `task_status.ts` (+197), substantial agent/instance refactor, and `instance-store.ts` (+200). This PR is effectively a branch sync masquerading as a one-line fix — reviewers cannot meaningfully gate the actual `${tmp}` change without auditing the entire surface.

2. **`finalize-latest-json.ts` (+144/-89)** and the new macOS `.app.tar.gz` packaging step in `publish.yml` are release-pipeline changes. Combined with removing the entire `build-tauri` job, this is a delivery-channel migration. It should be its own PR with a release-notes story; bundling it under "fix beta tmp" makes auditing the release surface impossible.

3. **New `repo_clone` tool (`packages/opencode/src/tool/repo_clone.ts`, +207)** introduces network egress capability. There's no visible scope check in the snippet around what URLs it can clone — needs a security review on its own. Bundling a new exfiltration-capable tool under a typo-fix PR is exactly the pattern that should not slip through.

4. **The actual `${tmp}` fix appears correct** — adding `tmp: Global.Path.tmp` to render values plus the `shell.test.ts` "tmp placeholder" assertion is the minimal correct change. If the author splits this into (a) tmp-fix + test, (b) build-pipeline migration, (c) new tools (repo_clone, repo_overview, task_status), each piece can land cleanly.

## Recommendation

Block on scope. The one-line fix is good; the surrounding 4500-line diff needs to be split before merge.
