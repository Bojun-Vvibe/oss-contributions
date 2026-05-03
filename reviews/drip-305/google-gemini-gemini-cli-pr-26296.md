# google-gemini/gemini-cli PR #26296 — fix(github): do not sync 'help wanted' label to PRs

- **Author:** spencer426
- **Head SHA:** `b18e4ae8370b791eb6d827193990f4497d7604a3`
- **Base:** `main`
- **Size:** +2 / -1 (1 file)
- **Fixes:** #26297

## What it does

Removes `help wanted` from the regex allowlist in
`.github/scripts/pr-triage.sh::get_issue_labels` so that the bot stops
copying that label from a linked issue onto a PR, and adds an inline
comment explaining the omission.

## Diff observations

`.github/scripts/pr-triage.sh:42-44`:

```diff
-    labels=$(echo "${gh_output}" | grep -x -E '(area|priority)/.*|help wanted|🔒 maintainer only' | tr '\n' ',' | sed 's/,$//' || echo "")
+    # Note: 'help wanted' is intentionally omitted here so it does not propagate from issues to PRs.
+    labels=$(echo "${gh_output}" | grep -x -E '(area|priority)/.*|🔒 maintainer only' | tr '\n' ',' | sed 's/,$//' || echo "")
```

Single regex change plus a one-line guard comment. `area/*`, `priority/*`,
and `🔒 maintainer only` are still synced.

## Strengths

- The intent is captured by a comment in the file, not just the PR
  description — future contributors won't re-add `help wanted` "for
  consistency".
- Uses `grep -x` so the alternation can't accidentally match a longer label
  containing the substring "help wanted".
- Pure script change; no runtime risk to the CLI itself.

## Concerns

- No test coverage for this script. The repo presumably has bash-script
  smoke tests somewhere; if so, an extra fixture (issue with `help wanted`
  → PR with no `help wanted`) would be cheap. If not, fine.
- The PR description claims `bash -n` validation; that only checks syntax,
  not semantics. A quick local run with a fake `gh_output` containing
  `help wanted` would have been worth showing.

## Nits (non-blocking)

- Trailing whitespace on the line right after the new regex is preserved
  from the original (the `sed 's/,$//' || echo ""` line); not introduced
  by this PR.

## Verdict

**merge-as-is**
