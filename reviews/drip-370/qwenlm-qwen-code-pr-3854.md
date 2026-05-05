# QwenLM/qwen-code PR #3854 — ci: add Qwen Code issue follow-up bot workflow

- URL: https://github.com/QwenLM/qwen-code/pull/3854
- Head SHA: `e886bf96b711b3111ae6e81ff10e242b16f02b51`
- Author: stacked draft (related to #3843 / #3851)
- Size: +1305 / -2 (4 files: `issue-followup-bot.mjs`, its test, two workflow YAMLs)

## Verdict
`needs-discussion`

## Rationale

The implementation is conservative and well-tested for what it does — the bot script at
`.github/scripts/issue-followup-bot.mjs:1-903` ships a real heuristics layer (stop-word list at `:13-39`,
`TRIVIAL_PHRASES` at `:42-59`, `MEANINGFUL_SHORT_TERMS` at `:62-77` so single-token bug reports like
`"oauth"` or `"cli"` aren't auto-closed as test placeholders, `USER_TEMPLATE_HEADINGS` at `:80-85`
that recognizes the actual `### What happened?` / `### Login Information` GitHub-template-rendered
headings rather than guessing), correct `<!-- qwen-issue-bot:* -->` HTML-comment markers at `:8-10`
to dedupe its own past comments and avoid talking over itself, sticky-comment update semantics, and
opt-in rollout via `QWEN_ISSUE_FOLLOWUP_BOT_ENABLED=true` repository variable so issue-event and
scheduled paths only become active after a maintainer flips the switch. The `MAX_GITHUB_REQUEST_ATTEMPTS = 3`
retry policy at `:14` and the `RELATED_ISSUE_LIMIT = 3` cap at `:13` are reasonable defaults. The
test file's coverage list (test-issue detection, short-gibberish template reports, missing-info
flow, related-issue matching, duplicate-comment suppression, weak-match suppression, label
inference, transient-API retry, scheduled-default-limit) is the right set of things to pin.

The reason this is `needs-discussion` rather than `merge-after-nits` is governance, not code:
this is a self-modifying-bot surface with closing power over OSS contributor issues, and three
specific decisions need explicit maintainer policy sign-off before it lands on the default branch
where the workflow becomes auto-triggerable: (1) **the auto-close criteria** — `hasOnlyShortTemplateNoise()`
at `:172-187` requires the *title* to be short-and-meaningless (no spaces, ≤12 char compact-signal,
not in `MEANINGFUL_SHORT_TERMS`) AND every template body value to be short-template-noise; this is
conservative but the policy of "when does a bot get to close a real human's report" is exactly the
governance call that a code reviewer can't make for the project. (2) **the bot identity** — the PR
correctly avoids `GITHUB_TOKEN` fallback so comments don't appear as `github-actions[bot]`, but
this means *every* comment / close action is signed by `QWEN_CODE_BOT_TOKEN` or `CI_BOT_PAT`, and
the audit trail attribution policy (and how the bot identifies itself in the comment body so
reopens are friction-free for the affected user) needs to be a maintainer call. (3) **the
default `scheduled_limit=10`** at `:12` — manual-dispatch UX is "user types `1` for a small
canary", but the *scheduled* path runs unattended at `scheduled_limit=10` until someone changes it;
a one-time bootstrap value of `1` for the first N scheduled runs after merge would be safer than
defaulting to 10.

Concrete asks before merge: (a) split into two PRs — the script + tests + workflow YAMLs without
the `schedule:` trigger first (so it's manual-dispatch-only), then a follow-up that turns on
scheduled processing once the first round of dry-runs is reviewed; (b) add a one-line code comment
at `:172` linking to the maintainer-approved policy doc that defines "auto-closeable" criteria, so
future edits to `hasOnlyShortTemplateNoise` cannot drift policy silently; (c) the `RELATED_ISSUE_LIMIT = 3`
matching threshold uses `STOP_WORDS` for token filtering but I don't see a Jaccard / cosine
threshold pinned in the diff excerpt — the related-issue comment is the only public wrong-positive
the bot can throw, so the actual numeric similarity floor that gates a related-issue comment should
be a named constant with a comment defending the value, not buried in `findRelatedIssues`.
