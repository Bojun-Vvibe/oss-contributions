# google-gemini/gemini-cli PR #25978 — point plan-mode session retention docs at actual /settings labels

- **PR**: https://github.com/google-gemini/gemini-cli/pull/25978
- **Author**: @ifitisit
- **Head SHA**: `270d16af6fe312e0ba1c6cfc681a1abe916ef246`
- **Size**: +2 / −1
- **Files**: `docs/cli/plan-mode.md`

## Summary

Two-line docs fix. The plan-mode session-retention paragraph instructed users to "search for **Session Retention**" in the `/settings` UI, but no setting with that label exists. The settings panel actually exposes **Enable Session Cleanup** and **Keep chat history**. The PR rewords the docs to point at the real labels.

## Verdict: `merge-as-is`

A docs-only fix where the existing instruction was actively wrong (the label literally did not exist in the settings UI). The new wording is honest about there being two related toggles, which matches the actual UI. Net cost: 1 line added. Net benefit: every user who searched "Session Retention" in `/settings` and found nothing now gets the right hint.

## Specific references

- `docs/cli/plan-mode.md:472-475` — old: `(search for **Session Retention**)`. New: `(search for **Enable Session Cleanup** or **Keep chat history**)`. The `or` is the right connective because the two settings govern adjacent but distinct behavior (cleanup-on-exit vs. how-many-days-to-keep), and a user looking for "session retention" might want either.
- The link to `../cli/session-management.md#session-retention` is preserved unchanged. That anchor exists in the canonical reference, so users who follow the link still land in the right place.
- The phrasing keeps the leading article-vs-no-article cadence (`/settings command` and `settings.json file`) consistent with the surrounding paragraph — no copy-edit drift.

## Nits

None. The PR is two lines and does exactly what its title says. If anything, a forward-looking improvement would be a CI lint that diffs setting-name labels in `*.md` against the strings rendered by the settings panel — but that's well outside the scope of a docs fix.

## What I learned

Docs that name UI labels by string are a classic source of low-grade drift: the settings panel gets renamed in a UI sprint, and the docs don't get updated because the writer doesn't grep for the old label. The honest fix is what this PR does — name the *current* labels — but the durable fix is to lint for `**Setting Name**`-shaped strings in docs against an extracted list of real labels. Worth filing as a follow-up issue rather than expanding this PR's scope.
