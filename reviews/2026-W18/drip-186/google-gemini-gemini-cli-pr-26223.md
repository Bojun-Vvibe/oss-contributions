---
pr: google-gemini/gemini-cli#26223
sha: bf78e08ba58588d36cd40e397875d891aeae6494
verdict: merge-after-nits
reviewed_at: 2026-04-30T00:00:00Z
---

# ci(github-actions): switch to github app token and fix bot self-trigger

URL: https://github.com/google-gemini/gemini-cli/pull/26223
Files: `.github/workflows/gemini-cli-bot-brain.yml`
Diff: 24+/12-

## Context

Two related fixes to the bot-brain workflow:

1. **Auth migration**: replaces a long-lived PAT
   (`secrets.GEMINI_CLI_ROBOT_GITHUB_PAT`) with on-demand GitHub App
   tokens minted via `actions/create-github-app-token@v2` at
   `gemini-cli-bot-brain.yml:193-202` (pinned by SHA + ratchet
   comment). The minted token at `steps.generate_token.outputs.token`
   then replaces the PAT at the two `GH_TOKEN:` env declarations
   (`:220` and `:262`). Git committer identity also flips from
   `gemini-cli-robot` (PAT-bound human-style account) to
   `gemini-cli[bot]` / `gemini-cli[bot]@users.noreply.github.com`
   (the canonical GitHub App identity format) at `:223-224`.
2. **Self-trigger guard**: the workflow trigger condition at `:44`
   gains a `github.event.comment.user.login != 'gemini-cli[bot]'`
   clause. The mention text also flips from `@gemini-cli-robot` to
   `@gemini-cli` to match the new bot handle, and the deleted PR-author
   safety check at `:268-273` is the workflow accepting that App-posted
   PRs won't have the literal `gemini-cli-robot` author string anymore.

The comment-posting layer also switches from `gh issue comment` /
`gh pr comment` (which use GraphQL under the hood and can hit
identity/auth edge cases with App tokens) to direct REST API calls
via `gh api repos/.../issues/{n}/comments -F body=@...` at `:268-269`
and `:277-278`.

## What's good

- Action pinned by full SHA `a8d616148505b5069dccd32f177bb87d7f39123b`
  at `:196` with the `# ratchet:actions/create-github-app-token@v2`
  comment is the load-bearing supply-chain detail — third-party
  Actions should always be SHA-pinned and the ratchet annotation
  makes auto-update tooling honest about the version semantics.
- Token minted with explicit narrow permissions at `:202`:
  `{"contents": "write", "pull_requests": "write", "issues":
  "write", "workflows": "write"}`. This is least-privilege — the
  prior PAT had whatever scopes the bot account was issued with
  (typically `repo` scope = ~all-the-things).
- The `if:` gate on the token-minting step at `:195` matches the
  exact set of triggers that need the token (the `enable_prs` /
  `issue_comment` / `run_interactive` triple), so unrelated
  scheduled/dispatch invocations don't burn a token mint they don't
  need.
- The self-trigger guard at `:44` is the right shape: a `!=
  'gemini-cli[bot]'` check on the canonical bot identity, evaluated
  before the mention-substring check, so the bot can't be tricked
  into recursive triggering by quoting its own message in a reply.
- Switching to REST API at `:269,278` is the correct workaround for
  the documented App-token + GraphQL friction with `gh issue/pr
  comment`. The `-F body=@<file>` form correctly handles multi-line
  markdown comments without shell-escaping issues.

## Nits / follow-ups

- The deletion of the PR author safety check at `:268-273` (the prior
  `if [ "$PR_AUTHOR" != "gemini-cli-robot" ]; then ... exit 1; fi`)
  is necessary for the migration but loses a real safety net. The
  original check was preventing the bot from commenting on a PR it
  didn't author (the "safety abort" message at the deleted code).
  The replacement comment at `:271-272` ("Skipping author validation
  here to let the app post.") accurately describes what was deleted
  but doesn't replace the protection. Suggested replacement: check
  that `PR_AUTHOR == 'gemini-cli[bot]'` (the new identity), keeping
  the abort behaviour with the new identity string.
- The `app-id` and `private-key` secrets at `:200-201` need rotation
  documentation — App private keys are forever unless explicitly
  rotated, and the prior PAT presumably had an expiration policy.
  A README note at the workflow or in `SECURITY.md` naming the
  rotation cadence would close that gap.
- The mention text change `@gemini-cli-robot` → `@gemini-cli` at `:44`
  is a user-visible breaking change for anyone with muscle memory or
  documentation referencing the old handle. Worth a CHANGELOG /
  CONTRIBUTING.md update at minimum, and ideally a transition period
  where both substrings are accepted.
- The `permissions:` job-level block at `:188-191` declares
  `pull-requests: 'write'` and `actions: 'write'`, but the App-minted
  token is what's actually used for the API calls now — the workflow
  job's `GITHUB_TOKEN` permissions are mostly cosmetic. Worth either
  reducing the job-level permissions (since the App token does the
  writes) or documenting why both are needed.

## Verdict

`merge-after-nits` — correct migration off a long-lived PAT to
short-lived App tokens with proper SHA-pinning and least-privilege
permissions, plus a real self-trigger fix. The deleted PR-author
safety check is the one item that should be replaced rather than just
removed before this lands, since the comment-poster step is one of
the higher-blast-radius things the bot does.
