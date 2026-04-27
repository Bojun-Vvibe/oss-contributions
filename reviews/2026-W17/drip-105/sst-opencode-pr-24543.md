---
pr: https://github.com/sst/opencode/pull/24543
sha: bb6a9353
diff: +22/-8
state: OPEN
---

## Summary

Hardens the session-route effect against stale-closure writes when the user switches sessions mid-fetch, and bolts on an unrelated kv-toggle (`sync_prompt_context_on_session_switch`) plus two scrollbar color tweaks and an empty `packages/sdk/js/openapi.json`. The core fix in `routes/session/index.tsx` is correct; everything else is scope creep that should be split.

## Specific observations

- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:185-211` — captures `route.sessionID` once into `currentSessionID`, then guards every post-await write with `if (route.sessionID !== currentSessionID) return`. This is the right shape for SolidJS `createEffect` + async work: the effect can re-run before `sdk.client.session.get` resolves, and previously the late `result` would be applied to whatever the *new* session is now. The guard at `:189` (after `session.get`) and at `:213` (after `sync.session.sync`) cover both await boundaries; the catch handler at `:217` already had the guard, so all three exit paths are now consistent.
- `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx:241-242` — `const shouldSync = kv.get("sync_prompt_context_on_session_switch", true); if (!shouldSync) return;` early-exits the agent/model copy on session switch. The default `true` preserves current behavior, which is good. But the variable name reads as a boolean-on-by-default ("do sync"), while the command-palette label at `routes/session/index.tsx:687` reads `"Keep model/agent when switching sessions"` when the flag is **on** — that wording suggests the *current* session's model is kept (sticky), when in fact the flag controls whether the *previous* session's prompt context is **copied into** the new one. Naming and label disagree; pick one mental model.
- `routes/session/index.tsx:1077-1080` — scrollbar `backgroundColor: theme.background` + `foregroundColor: theme.borderActive` is a visual change with no code-level justification in the diff. The PR title says "guard workspace mutation against stale session effect" — these belong in a separate PR or at minimum a separate commit with rationale, otherwise reviewers can't tell whether the new colors regress accessibility/contrast on any of the 30+ themes shipped.
- `packages/sdk/js/openapi.json` is created as a 0-byte file. Either the generator was supposed to populate it and didn't, or it's accidental. Either way an empty `openapi.json` checked into a published SDK package is a footgun: downstream tooling that does `JSON.parse(read("openapi.json"))` will throw on the next consumer install. Drop the file or generate it for real.
- The kv-signal/kv-get split between the prompt component (uses `kv.get`, no reactivity) and the session route (uses `kv.signal`, reactive) is intentional — the prompt only reads the flag inside the effect that fires *on* session change, so a one-shot read is fine. Worth a one-line comment so a future reader doesn't "fix" it to `kv.signal` and introduce a re-render storm.

## Verdict

`request-changes` — the stale-effect guard is the right fix and merge-ready in isolation, but this PR is three unrelated changes in one trench coat. Split into: (1) the session-effect guard (this is the titled fix), (2) the `sync_prompt_context_on_session_switch` toggle with a clearer label/name pair, (3) the scrollbar color change with a screenshot and contrast rationale. Drop the empty `openapi.json` either way.

## What I learned

SolidJS `createEffect` re-runs on dep change but does not cancel its in-flight async work — the canonical pattern is "capture deps into locals, re-check the live signal after every `await`, bail on mismatch." The `try/catch` at the outer IIFE boundary needs the *same* guard or stale-error toasts leak across navigations.
