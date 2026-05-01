# google-gemini/gemini-cli#26296 — fix(github): do not sync 'help wanted' label to PRs

- Repo: google-gemini/gemini-cli
- PR: https://github.com/google-gemini/gemini-cli/pull/26296
- Head SHA: `b18e4ae8370b`
- Size: +2 / -1 (one file)
- Verdict: **merge-as-is**

## What the diff actually does

One regex change in `.github/scripts/pr-triage.sh`, inside the
`get_issue_labels()` function around line 42–44.

Before:
```bash
labels=$(echo "${gh_output}" | grep -x -E '(area|priority)/.*|help wanted|🔒 maintainer only' | tr '\n' ',' | sed 's/,$//' || echo "")
```

After:
```bash
# Note: 'help wanted' is intentionally omitted here so it does not propagate from issues to PRs.
labels=$(echo "${gh_output}" | grep -x -E '(area|priority)/.*|🔒 maintainer only' | tr '\n' ',' | sed 's/,$//' || echo "")
```

Only the `|help wanted` arm of the alternation is dropped, plus the
preceding comment line is added. Everything else — the `grep -x -E`
flags, the `(area|priority)/.*` arm, the `🔒 maintainer only` arm, the
`tr` / `sed` post-processing, the fallback `|| echo ""` — is byte-for-
byte preserved.

## Why merge-as-is

- **Semantic shape is exactly right.** "help wanted" is a triage signal
  for *issues* ("a maintainer would welcome an external contributor
  taking this on"), not for *PRs* (which by definition already have
  someone working on them). Auto-syncing it from a linked issue to its
  PR is meaningless at best and misleading at worst — it tells external
  contributors "this PR needs help" when in reality a contributor is
  already mid-flight.
- **The comment is the load-bearing part.** Without the
  `# Note: 'help wanted' is intentionally omitted...` line, a future
  contributor doing a "I added a new label, let me extend the regex"
  PR would mechanically re-add `help wanted` thinking it was an
  oversight. The comment names the *intent* so the deletion can't be
  silently reverted.
- **Zero risk surface.** The `grep -x` (full-line match) flag means the
  regex change cannot accidentally drop a substring-overlapping label;
  removing one alternation arm removes exactly one label class.
- **Closes a referenced issue (#26297)** that presumably has the
  reproducer / motivating discussion.

## Nits (none blocking)

- The bash `grep -E` alternation could be moved to a named bash array
  (`SYNCED_LABELS=("area/.*" "priority/.*" "🔒 maintainer only")`) so
  future label additions/removals are one-line array edits with
  per-line comments, but that's a refactor, not a fix.
- The `|| echo ""` fallback at the end masks a `grep` exit-with-error
  case (input was empty), which is fine for this script's intent but
  worth a note if anyone ever wants to distinguish "no matching labels"
  from "input read error."

## Theme tie-in

Fix the label-pollution at the *allow-list source* (one regex arm,
one comment) rather than at every consumer ("filter help-wanted out
when displaying PR labels"). Source-of-truth fixes scale to every
downstream view; consumer fixes leave the next view broken.
