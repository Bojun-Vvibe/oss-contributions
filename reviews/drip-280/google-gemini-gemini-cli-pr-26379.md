# Review: google-gemini/gemini-cli #26379 — docs: fix GitHub capitalization in releases guide

- **PR**: https://github.com/google-gemini/gemini-cli/pull/26379
- **Head SHA**: `226b0ad7bb1abd116c6c17b5229142e15fdadf9a`
- **Diff size**: +2 / -2 lines, 1 file (`docs/releases.md`)

## What it changes

Two replacements in `docs/releases.md`:

- Line 11: `"Github-hosted NPM repository"` → `"GitHub-hosted NPM repository"`.
- Line 22 (table header): `"Github Private NPM Repo"` → `"GitHub Private NPM Repo"`.

That's it. Pure docs typo fix.

## Assessment

GitHub's own brand guidelines specify `GitHub` with a capital `H`. The releases doc had
two stragglers; this PR fixes them. No other occurrences of "Github" remain in the
modified file.

Worth a quick `grep -rn "Github" docs/` across the rest of the docs tree to see if
there are sibling stragglers that could be batched in. From the title and diff, the
author chose to scope tightly to `releases.md` rather than do a sweep — that's a
defensible choice (smaller blast radius, easier to review) but means a follow-up PR
may be needed for full cleanup.

No risk surface. No code paths touched. No build/test impact.

## Verdict

`merge-as-is` — pure docs typo, brand-correct, zero risk. Optional: a sweep of remaining
"Github" occurrences across `docs/` in a follow-up PR.
