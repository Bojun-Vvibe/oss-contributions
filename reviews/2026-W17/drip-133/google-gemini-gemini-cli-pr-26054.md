# google-gemini/gemini-cli #26054 — fix(cli): rename `adminControlsListner` -> `adminControlsListener`

**Verdict:** `merge-as-is`

- Repo: `google-gemini/gemini-cli`
- PR: https://github.com/google-gemini/gemini-cli/pull/26054
- Head SHA: `d622a078`
- Closes: #21803
- Diff: +4 / -4, single file `packages/cli/src/gemini.tsx`

## What it does

Pure typo fix: renames the local variable `adminControlsListner` →
`adminControlsListener` at the four call sites in `gemini.tsx`. The
function it returns from (`setupAdminControlsListener`, declared at
`gemini.tsx:805`) is already correctly spelled, so the misspelling was
never load-bearing — just visually grating and a frequent grep miss for
anyone trying to find listener-related code.

## Design analysis

Right shape and zero blast radius. The four touched lines are:

- `gemini.tsx:254-255`: declaration + `registerCleanup(adminControlsListener.cleanup)`
- `gemini.tsx:395`: `adminControlsListener.setConfig(partialConfig)`
- `gemini.tsx:529`: `adminControlsListener.setConfig(config)`

All four are local-scope references to the same `const`. The variable does
not cross module boundaries — `setupAdminControlsListener` is the imported
name, and that import is unchanged. So this PR cannot break:

- External consumers (none — variable is function-local)
- Type checking (TypeScript resolves the new name identically)
- Snapshot tests (variable name doesn't appear in any rendered output)
- Logging (variable name not used in any log string in the diff)

The diff is mechanical and complete: `git grep adminControlsListner
packages/cli/src/gemini.tsx` post-PR returns zero matches, which is the
right verification.

## Risks

Effectively none. Two micro-concerns worth mentioning for completeness:

1. **Other files may still spell it the wrong way.** The PR scopes itself
   to `gemini.tsx`. If `adminControlsListner` (with the typo) appears in
   any other file in the repo — comments, log strings, doc files, test
   names — those won't be touched. A `grep -rn "Listner" packages/`
   pre-merge sweep is worth it. Based on the closing-issue text and the
   fact that the function `setupAdminControlsListener` was already
   correctly spelled, the typo was likely localized to this one consumer,
   but it's a 5-second check.
2. **Closing #21803 may need confirmation.** If that issue's scope was
   broader than this one file, the issue-close on merge could leave
   loose ends. A note from the author confirming "this issue refers to
   exactly these four lines" would tighten that.

## Suggestions

- Run `git grep -in "listner" .` once before merge and either fold any
  hits into this PR or open a follow-up. The cost of the sweep is below
  the cost of opening a second typo-fix PR next week.
- No code suggestions for the diff itself — it's correct as-is.

## What I learned

Pure rename PRs in a monolithic config-loading function are the safest
class of change in any codebase. The right review investment is verifying
*scope*: did the author find every occurrence, or just the ones that
caused the bug they were chasing? `grep -rn` against the misspelling is a
30-second exercise that should be part of any rename PR's checklist.
Verdict-as-is is justified here because even if there are stragglers
elsewhere, the diff makes the file it touches strictly better.
