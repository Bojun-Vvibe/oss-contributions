# block/goose #8943 — fix: slow UI when trying to start a new project session

- PR: https://github.com/block/goose/pull/8943
- Author: matt2e
- Head SHA: `974d73250a0612326ab90d72ed6d96f3001224c6`
- Updated: ~2026-05-02

## Summary
The "New Session" button in the project view took a very long time to render — the previous implementation awaited an ACP `prepareSession` round-trip *before* the chat UI was created. This PR creates a local draft chat session immediately when starting from a project, then performs ACP binding asynchronously through a new `sessionStore.prepareSessionBinding(...)` chokepoint. Centralizes binding flow and preserves local-draft sessions across ACP refreshes.

## Observations
- `ui/goose2/src/app/AppShell.tsx:271-330`: the new `if (project)` early branch creates the local session synchronously via `sessionStore.createLocalSession({ title, projectId, providerId, modelId, modelName })`, sets active session + view, then returns. The old async `resolveSupportedSessionModelPreference` await is gone from this path. That's the actual UX fix — perceived latency drops to ~1 frame.
- `ui/goose2/src/app/AppShell.tsx:298-307`: model preference resolution uses the new `resolveSessionModelPreference` + `sanitizeSessionModelPreference` pair (synchronous, no provider-inventory probe). Confirm `sanitizeSessionModelPreference` falls back gracefully when `providerInventoryEntries.get(...)` returns `undefined` (cold start, inventory still loading) — otherwise the optimistic session could end up pinned to a missing model.
- `ui/goose2/src/app/AppShell.tsx:494-501`: the previously inline `acpPrepareSession(sessionId, providerId, workingDir, { personaId })` call is replaced with `sessionStore.prepareSessionBinding({ sessionId, providerId, workingDir, personaId, projectId })`. Centralizing this is the right architectural move — the `useChatSessionController` test added at `useChatSessionController.test.ts:226-302` relies on this chokepoint to bind a local session at model-pick time. Verify there are no remaining direct `acpPrepareSession` callers from other code paths (sidebar, resume picker, persona switch) that bypass the new chokepoint and re-introduce the optimistic-session-without-binding bug class.
- `useChatSessionController.test.ts:226`: the new "binds an optimistic project session before setting its model" test asserts the exact wire format — `acpPrepareSession("local-session-1", "openai", "/tmp/project", { personaId: undefined, projectId: "project-1", knownNew: true })`. Note the new `knownNew: true` — that flag isn't visible in this PR's `AppShell.tsx` diff snippet, so the call must be inside `prepareSessionBinding`. Confirm the binding store actually sets `knownNew: true` for local-draft sessions and `false` for resumed ones; otherwise ACP will treat resumed-but-unbound sessions as new and lose their history.
- The mock at `useChatSessionController.test.ts:95-110` adds `gpt-4.1` → `openai` mapping. Reasonable for a regression test fixture; not a production change.
- `findExistingDraft`-driven early return at `AppShell.tsx:289-296` (still present) means clicking "New Session" twice on a project with no chats returns the same local draft — that matches user expectation, but verify the second click also re-issues `prepareSessionBinding` if the first one's promise rejected (network blip during ACP cold start) or the session will sit in an unbindable state forever.
- No telemetry/perf assertion added for the new fast path. The author's `perfLog` call (`[perf:newtab] ... created local project session in X ms`) would be a natural place to assert the regression doesn't come back; consider a Vitest test that spies on `perfLog` and asserts the synchronous path doesn't await ACP.
- Pre-push checks reportedly pass (Biome, file-size/i18n, typecheck, build, Vitest, cargo fmt/check/clippy). Good — `goose2` UI plus rust workspace both green.

## Verdict
`merge-after-nits`
