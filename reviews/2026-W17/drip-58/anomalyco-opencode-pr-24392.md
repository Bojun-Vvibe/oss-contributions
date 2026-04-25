# anomalyco/opencode#24392 — chore: add changelog sync workflow and changelog

- **Repo:** anomalyco/opencode
- **Author:** moscovium-mc
- **Head SHA:** `a20420e7` (a20420e76659f8073c410a162689527a68e4c164)
- **Size:** +482 / -0

## Summary
Adds a scheduled GitHub Actions workflow (`.github/workflows/sync-changelog.yml`)
that hourly fetches `https://opencode.ai/changelog`, parses out a JS object
literal embedded in the HTML, regenerates `changelog.md`, and pushes to `dev`.

## Specific references
- `.github/workflows/sync-changelog.yml:4-6` @ `a20420e7` — `cron: "0 * * * *"` every hour. Scrapes a page that's controlled by the same org, so cadence is fine, but workflow runs are a non-trivial cost for a derivative artifact.
- `.github/workflows/sync-changelog.yml:33-46` @ `a20420e7` — the parser uses regexes (`/tag:"v([\d.]+)"/g`) against the HTML response. This is brittle: any minified-output reshuffle breaks parsing silently because the script writes an empty `changes.json` rather than failing.
- `.github/workflows/sync-changelog.yml:51-64` @ `a20420e7` — section parser substring-walks `title:"Core"` etc. with a fixed `secIdx < 500` window. Cliffhanging value choice; one extra wrapper element kills it.
- `.github/workflows/sync-changelog.yml:104-114` @ `a20420e7` — commit step runs `git diff --cached --quiet` before pushing. Good idempotency.

## Observations
1. **Silent parser failure**: if the HTML structure changes, `versionRegex` matches zero times and the workflow happily commits an empty changelog (`fetch-changelog.mjs` writes `[]`, generator emits just the header). At minimum, fail the job when `changes.length === 0` so a regression surfaces as a red CI run instead of a deletion commit.
2. **`actions/checkout` uses `secrets.GITHUB_TOKEN`** (line 22). Pushes to `dev` will be made by `github-actions[bot]` and won't trigger downstream workflows on `dev` (by GH design). If anything depends on those — release notifications, deploy-on-push — it will silently stop. Consider a PAT or a `repository_dispatch` fan-out.
3. **No fetch retry / error handling**: a transient 502 from `opencode.ai/changelog` causes the same empty-changelog scenario as #1. A simple `if (!res.ok) throw` plus retry-on-fail would harden this.

## Verdict
**merge-after-nits** — fail-loud on empty parse output is the only real blocker; the rest is hardening.
