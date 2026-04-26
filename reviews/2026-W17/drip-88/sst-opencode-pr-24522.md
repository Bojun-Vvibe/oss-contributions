---
pr: 24522
repo: sst/opencode
sha: bb6a93539710fb9d7cd413aa072624b60d8a5900
verdict: request-changes
date: 2026-04-27
---

# sst/opencode #24522 — fix: guard workspace mutation against stale session effect

- **Author**: alfredocristofano
- **Head SHA**: bb6a93539710fb9d7cd413aa072624b60d8a5900
- **Size**: +22/-8 across `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx`, `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx`, `packages/sdk/js/openapi.json`.

## Scope

The PR title/body promise a single race-condition fix: capture `route.sessionID` into `currentSessionID` before the `await sdk.client.session.get(...)`, then bail with `if (route.sessionID !== currentSessionID) return` after the await before mutating `project.workspace` / calling `sync.bootstrap()`. That core fix is correct and small — but the diff carries **at least two unrelated, undisclosed changes** that don't belong in a "guard against stale effect" PR.

## Specific findings

- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:185-220` — the actual race fix. `currentSessionID = route.sessionID` snapshot + `if (route.sessionID !== currentSessionID) return` post-await is exactly right; renaming all in-effect references to `currentSessionID` (the toast message, both `sync.session.sync` calls, the catch-handler guard) is the consistent shape. **Approve this slice on its own merits.**
- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:168` and `:687-695` — adds a brand-new `sync_prompt_context_on_session_switch` `kv.signal` plus a "Keep model/agent when switching sessions" command palette entry. This is a **separate user-facing feature**, not a race-fix. PR #24508 ("feat: toggle to keep model/agent when switching sessions") already exists for exactly this. Either #24522 is being used to land #24508 stealthily, or the author rebased on top of #24508 and forgot to drop the slice — either way it shouldn't ship under the title "guard workspace mutation against stale session effect".
- `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx:241-243` — the matching consumer side of that toggle (`if (!shouldSync) return` based on the new kv key). Same problem as above: belongs in #24508, not here.
- `packages/opencode/src/cli/cmd/tui/routes/session/index.tsx:1077-1079` — silently swaps the scrollbar palette: `backgroundColor: theme.backgroundElement → theme.background` and `foregroundColor: theme.border → theme.borderActive`. PR #24507 ("fix: make scrollbar visible in Monokai theme") owns this exact change. Same diagnosis: rebase leakage or stealth-ship.
- `packages/sdk/js/openapi.json` — added as an empty file (`new file mode 100644 ... index e69de29bb2d`, 0 bytes). Either generator output that was supposed to be regenerated and got committed empty, or .gitkeep-ish placeholder. Either way an empty JSON file in the SDK package will surprise the next person who runs the OpenAPI generator. Drop it.

## Risk

The race-fix slice itself is low-risk and lands a real correctness bug (workspace state being mutated for the wrong session id). The bundled toggle + palette tweak risks (a) surprising reviewers of the named PRs (#24507, #24508) when they merge first and conflict, and (b) shipping a settings key (`sync_prompt_context_on_session_switch`) without tests, docs, or default-behavior justification. The empty `openapi.json` risks confusing the SDK regen pipeline.

## Verdict

**request-changes** — split the diff. Keep this PR scoped to the `currentSessionID` guard + matching catch handler (the +13 net lines that match the title). Move the `sync_prompt_context_on_session_switch` signal/handler/palette-entry to PR #24508. Move the scrollbar color swap to PR #24507. Drop the empty `openapi.json` or replace with the actual generated content. The race fix on its own would be `merge-as-is`; the rebase contamination is what holds this back.
