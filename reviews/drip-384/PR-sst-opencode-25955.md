# sst/opencode#25955 — fix: find GitHub remote from any remote, not just origin

- **Head SHA**: `99577a89d661d17eb7a8fb2900909f499e5cbe38`
- **Stats**: +12 / -4, 1 file (`packages/opencode/src/cli/cmd/github.ts`)

## Summary

`GithubInstallCommand` was hard-coded to read `git remote get-url origin`, so users whose GitHub remote was named `upstream` (forked workflows), `github` (multi-host setups with a separate `origin` pointing at GitLab), or anything else got an unhelpful "Could not find git repository" error and had to rename their remotes to install. This PR widens the lookup to `git remote -v`, parses each URL, and picks the first one that successfully parses as a GitHub remote.

## Specific citations

- `packages/opencode/src/cli/cmd/github.ts:264-275`: swaps `["remote", "get-url", "origin"]` for `["remote", "-v"]` and splits the output by newline, taking the second whitespace-delimited column. Correct shape: `git remote -v` emits `<name>\t<url> (fetch|push)` per line, so `line.split(/\s+/)[1]` is the URL. The `.filter(Boolean)` guards against trailing blank lines.
- `:276`: `urls.map(parseGitHubRemote).find((p): p is { owner: string; repo: string } => p !== null)` — type predicate is correct, picks the first parseable GitHub URL across all remotes (fetch + push entries for every remote get duplicated by `git remote -v` so this naturally dedupes by short-circuiting at the first hit).
- `:278-280`: error message updated from "Could not find git repository" to "Could not find GitHub remote" — more accurate now that the precondition is "no GitHub URL among any remote" rather than "no git repo".

## Verdict

**merge-after-nits**

## Rationale

The fix is correct and the new error message is sharper. Two minor nits worth flagging before merge: (1) the precedence is "first remote alphabetically that parses" because `git remote -v` orders by remote name; this means if a user has both `origin` (a GitLab fork) and `upstream` (the actual GitHub source), upstream wins — likely the desired behavior but worth a comment naming the precedence; (2) duplicate entries (fetch + push lines for the same remote) get re-parsed redundantly — harmless for correctness but a `new Set(urls)` would be a 5-character speedup. No tests added — a small unit test feeding a fixture `git remote -v` string with mixed `origin=gitlab, upstream=github` to assert the right URL wins would harden the precedence guarantee and prevent regressions if someone later swaps `find` for `filter()[0]` (which would have different behavior on mixed-host configs after a future refactor).
