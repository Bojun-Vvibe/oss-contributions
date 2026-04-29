---
pr: google-gemini/gemini-cli#26220
sha: 8c3aab81a42c5a79cedcda43208a91d95c51e4fa
verdict: merge-as-is
reviewed_at: 2026-04-30T00:00:00Z
---

# fix(core): discourage unprompted git add . in prompt snippets

URL: https://github.com/google-gemini/gemini-cli/pull/26220
Files: `packages/core/src/prompts/snippets.ts`, `packages/core/src/prompts/snippets.legacy.ts`, `packages/core/src/core/__snapshots__/prompts.test.ts.snap`

## Context

The "git repo workflow" prompt snippet is injected into the system prompt
whenever the agent runs in a git repo. It told the model to use
`git add ...` without distinguishing path-specific staging from the
catch-all `git add .` / `git add -A`. The result, observable in real
sessions, is that the agent sometimes stages unrelated dirty files (build
output, scratch files, accidentally-untracked secrets) along with the
intended change. This is a recurring footgun for both end users and CI
runs that operate on shared workspaces.

## What changed

Two textual edits to the same prompt block, applied symmetrically in both
`snippets.ts` (current) and `snippets.legacy.ts` (legacy renderer), with
the matching snapshot update:

1. `git add ...` → `git add <file>...` for specific files as needed.
2. New bullet immediately after: "Do not use `git add .` or `git add -A`
   unprompted as this can stage unwanted or untracked files. Instead,
   stage only the specific files that were changed or created as part of
   the task."

Snapshot at `prompts.test.ts.snap:1340-1346` confirms the rendered
output picks up both edits.

## What's good

- Edit applied to both `snippets.ts` and `snippets.legacy.ts` — the
  classic miss here is to update the active path and forget the legacy
  one, which silently leaves half of users on the buggy prompt. This PR
  catches both.
- Snapshot updated in the same commit, so the test suite stays green
  without a follow-up.
- Phrasing is conservative ("unprompted") rather than absolute, which
  preserves the legitimate "user explicitly asked me to stage everything"
  path.

## Verdict

`merge-as-is` — prompt-engineering fix to a documented behavior problem,
applied to both render paths, snapshot kept in sync. No code-path risk.
