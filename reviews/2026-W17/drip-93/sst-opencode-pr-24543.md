---
pr: 24543
repo: sst/opencode
sha: bb6a93539710fb9d7cd413aa072624b60d8a5900
verdict: merge-after-nits
date: 2026-04-27
---

# sst/opencode #24543 — fix: guard workspace mutation against stale session effect

- **Author**: alfredocristofano
- **Head SHA**: bb6a93539710fb9d7cd413aa072624b60d8a5900
- **Link**: https://github.com/sst/opencode/pull/24543
- **Size**: +22/-8 across `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx`, `.../routes/session/index.tsx`, plus a stray empty `packages/sdk/js/openapi.json`.

## Scope

Closes #24542. A `createEffect` in the session route awaits `sdk.client.session.get()`; if the user navigates to a different session before the await resolves, the continuation mutates `project.workspace` for the *current* session using data fetched for the *previous* session. Fix captures `currentSessionID = route.sessionID` before the await and short-circuits with `if (route.sessionID !== currentSessionID) return` after each suspension point. Also adds a `sync_prompt_context_on_session_switch` kv toggle (default `true`) that lets users opt out of the agent/model-sync side of the same effect.

## Specific findings

- `routes/session/index.tsx:185` — `const sessionID = route.sessionID` renamed to `const currentSessionID = route.sessionID`. Pure naming clarity, fine.
- `routes/session/index.tsx:189` — new guard `if (route.sessionID !== currentSessionID) return` inserted *immediately* after `await sdk.client.session.get(...)`. This is the core fix and is in the right place.
- `routes/session/index.tsx:212-215` — `await sync.session.sync(currentSessionID)` and the subsequent `route.sessionID === currentSessionID` scroll guard both use the captured ID. The catch-handler at `:218` also uses `currentSessionID` for its early-return guard. Consistent.
- **Nit**: the second await at `:212` (`await sync.session.sync(currentSessionID)`) does *not* re-check `route.sessionID !== currentSessionID` before the scroll-by-100k call at `:213`. The check is there (`route.sessionID === currentSessionID && scroll`) but only to gate the scroll, not the upstream `sync.session.sync` call itself. If the user switches away during the bootstrap phase at `:208-211`, `sync.session.sync` will still execute against `currentSessionID` — probably fine because `sync.session.sync(id)` is idempotent and ID-scoped, but worth noting in a comment.
- `component/prompt/index.tsx:241-243` — new `kv.get("sync_prompt_context_on_session_switch", true)` early-return so the agent/model snap-on-switch behaviour can be turned off. Reasonable; the keybind for the toggle is wired at `routes/session/index.tsx:687-695` under category `"Session"`.
- `routes/session/index.tsx:1077-1079` — unrelated cosmetic change to scrollbar trackOptions (`backgroundColor: theme.background`, `foregroundColor: theme.borderActive`). Should be a separate PR; bundling it muddies the bisect history for the actual race-fix.
- `packages/sdk/js/openapi.json` — a brand-new **empty** file (0 bytes) shows up in the diff. This is almost certainly an artifact of a local build step (codegen wrote a placeholder) that should not be committed. Even if it's intentional, a 0-byte JSON file is malformed; either remove it or write `{}`.

## Risks

The race-fix itself is low-risk and correct. The two unrelated changes (scrollbar theme, empty openapi.json) raise the review burden and are the reason this isn't a clean merge.

## Verdict

**merge-after-nits** — the race-condition fix is solid and in the right place, but the PR also drops a 0-byte `openapi.json` and an unrelated scrollbar restyling. Author should drop the empty file and split the scrollbar change out, or at minimum document them in the commit message.
