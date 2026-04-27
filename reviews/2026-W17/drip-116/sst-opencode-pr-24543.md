# sst/opencode#24543 — fix: guard workspace mutation against stale session effect

- PR: https://github.com/sst/opencode/pull/24543
- Head SHA: `bb6a9353`
- Diff: +22 / -8 across `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx` (+3), `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx` (+19/-8), and a new empty `packages/sdk/js/openapi.json`

## What it does
Closes #24542. Inside the session route's `createEffect` (`routes/session/index.tsx:185`), captures `route.sessionID` into a local `currentSessionID` *before* the `await sdk.client.session.get({ sessionID })` call, and adds `if (route.sessionID !== currentSessionID) return` *after* the await but *before* any workspace-state mutation (`:189`). All references inside the effect body and the catch handler are switched to `currentSessionID`. Without this guard, a user navigating away (or rapidly switching between sessions) during the in-flight session fetch could land workspace state — `previousWorkspace`, scroll position, sync handle — keyed to the *new* session but populated from the *old* session's data.

## Strengths
- Textbook async-stale-closure fix. Capture-before-await + identity-check-after-await is the canonical Solid/React pattern for this race, and the variable rename to `currentSessionID` makes it visually obvious which `sessionID` the closure is bound to.
- The guard fires *before* `previousWorkspace = project.workspace.current()` and `await sync.session.sync(currentSessionID)` (`:209`), so neither workspace state nor sync activity leaks into the wrong session.
- The catch handler at `:215` is correctly updated to compare against `currentSessionID` (was previously `sessionID`), ensuring stale errors from the old fetch don't surface as toasts in the new session.
- Pulls in a related KV-toggle (`sync_prompt_context_on_session_switch`) at `prompt/index.tsx:241-242` and `routes/session/index.tsx:168` that lets users opt out of the model/agent sync-on-switch behavior, with a corresponding command-palette entry at `:687-695`. Sensible UX addition for users who want session-pinned model/agent.

## Concerns
- **Scope creep.** The PR title and description say "guard workspace mutation against stale session effect" — but the diff also lands (a) the `sync_prompt_context_on_session_switch` KV toggle + command-palette entry, (b) a scrollbar theme tweak (`backgroundElement → background`, `border → borderActive` at `:1078-1079`), and (c) a *new empty file* `packages/sdk/js/openapi.json`. The empty `openapi.json` is almost certainly a `git add .` accident — generators don't write zero-byte files intentionally, and an empty `openapi.json` will likely break SDK generation downstream. Should be removed before merge.
- The KV-toggle defaults to `true` (sync on session switch), which preserves current behavior — that's the right default. But there's no test pinning either branch (`shouldSync === false` skips the model/agent setter) and no in-app surface (settings UI) describing what the toggle does. Command-palette-only discoverability is fine for power users; documenting this in `--help` or settings would help.
- No regression test for the actual race fix. Constructing the race in a unit test is non-trivial (needs to fake `sdk.client.session.get` resolution timing + `route.sessionID` mutation between await points), but a Vitest `vi.useFakeTimers()` + manual promise resolution would do it. Without this, the next refactor that drops the guard re-introduces #24542.
- Minor: the captured-variable pattern is now inconsistent within the same effect block. The `await sync.session.sync(currentSessionID)` line uses the captured value, but the *check* at `:209` is `if (route.sessionID === currentSessionID && scroll)` — fine, but the comparison flips polarity from the early-return guard at `:189`. A reader needs to track which sense each comparison is in. Adding a top-of-effect comment like `// race-guard: re-check route.sessionID after each await` would help.
- The scrollbar theme change has nothing to do with the stale-session bug. It's a visual tweak that should be its own PR with a screenshot — a reviewer can't tell from the diff alone whether it's intentional or a copy/paste mistake from a theme refactor.

## Verdict
**merge-after-nits** — the core fix is correct and important, but the empty `openapi.json` should be removed (likely accidental), the scrollbar theme tweak should be split into its own PR, and the KV-toggle scope is at minimum worth calling out in the PR description so reviewers don't miss it. After splitting, this is a clean merge.

## What I learned
"Capture before await, check after await" is the entire pattern for async-stale-closure prevention in reactive frameworks. The cost is one extra variable and one extra `if`; the benefit is preventing a whole class of race conditions that only show up under fast user input. PRs that bundle race fixes with unrelated UI tweaks are harder to review and harder to bisect later — splitting them reduces both review surface and revert risk.
