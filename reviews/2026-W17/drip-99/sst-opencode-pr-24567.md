# sst/opencode#24567 — fix: ignore GitHub Actions changelog contributor

- **Head**: `aafb42e5a6f77365f43f68c1006560164be133ce`
- **Size**: +1/-1 across 1 file
- **Verdict**: `merge-as-is`

## Context

`script/raw-changelog.ts` enumerates community contributors for the changelog
by walking PRs and filtering out a small denylist of bot/automation accounts.
The current denylist (`["actions-user", "opencode", "opencode-agent[bot]"]`)
misses `github-actions[bot]` — the default identity GitHub Actions uses when
a workflow commits or opens a PR with the auto-provisioned `GITHUB_TOKEN`.
The result is that recent generated changelogs were "thanking" the
`github-actions[bot]` account, which is noise.

## Design analysis

### The change: `script/raw-changelog.ts:26`

```ts
-const bot = ["actions-user", "opencode", "opencode-agent[bot]"]
+const bot = ["actions-user", "github-actions[bot]", "opencode", "opencode-agent[bot]"]
```

Single string literal added. Sort-merged into the existing alphabetical-ish
position (between `actions-user` and the project-named bots), which is a nice
touch — keeps the diff minimal and the list scannable.

### Why this is the right value

`github-actions[bot]` is the canonical login string GitHub returns for the
default Actions identity in the REST API (`user.login` field on PRs/issues
authored by a workflow that uses the default `GITHUB_TOKEN`). It is not
`github-actions` (no suffix), `actions-bot`, or any of the half-dozen
near-misses you sometimes see in copy-pasted denylists. The `[bot]` suffix
is part of the actual login slug, so the literal here matches what
`gh api` / `octokit` returns.

### The denylist shape vs. a smarter check

The list is a flat string-membership filter. An alternative would be
"detect any login ending in `[bot]`", which would future-proof against new
automation accounts. But:

1. Two of the three existing entries (`actions-user`, `opencode`) don't
   match the `[bot]` pattern, so a suffix check alone wouldn't replace the
   list — it would have to be additive.
2. Real human contributors with `bot` in their handle exist
   (e.g. `imbot`, `orbot`); a naive `endsWith("[bot]")` is safer than
   a substring check, but explicit allow/deny is still less surprising
   for a script that touches the public-facing changelog.
3. This is a 30-line script, not a library. Optimizing for readability
   over cleverness is the right call here.

So the explicit-list approach is fine, and the fix is correctly scoped to
add the one missing literal.

### Testing

Body lists `bun script/raw-changelog.ts --help` as the test step. That's a
smoke test (the script imports cleanly), not a behavior test — there's no
unit test that asserts the denylist filters out `github-actions[bot]`. For
a 1-line change to a denylist constant, that's an acceptable trade-off; a
test would essentially re-state the literal and add a maintenance burden
without catching real regressions.

## Concerns / nits

None worth blocking on.

1. *Could* sort the list strictly alphabetically (`actions-user`,
   `github-actions[bot]`, `opencode`, `opencode-agent[bot]`) to make
   future additions trivially correct. Not necessary.
2. The constant could move to `.github/BOT_AUTHORS` next to
   `.github/TEAM_MEMBERS` (which is already file-sourced via
   `Bun.file(...).text()` further down) for symmetry. Out of scope for
   a hotfix.

## Verdict reasoning

One-line denylist addition fixing a directly observable noise bug in the
generated changelog. The literal is correct (matches GitHub's actual
`user.login` for default Actions identity). No risk surface. Smoke-tested
via `--help`. This is the kind of change that should land immediately.

## What I learned

The `github-actions[bot]` login slug is a small but recurring trap when
building "human contributors only" filters: the suffix `[bot]` is part of
the login as returned by GitHub's API (it's not just rendering metadata
in the UI). Filters written without checking actual API output tend to
miss it — code review of denylists should always include "did the author
verify the literal against `gh api repos/.../pulls?per_page=1`?" as a
sanity check.
