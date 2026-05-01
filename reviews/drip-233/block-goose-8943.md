# block/goose#8943 — fix: slow UI when trying to start a new project session

- **Author:** Matt Toohey (matt2e)
- **Head SHA:** `974d73250a0612326ab90d72ed6d96f3001224c6`
- **Base:** `main`
- **Size:** +495 / -57 across 9 files
- **Files changed:** `ui/goose2/src/app/AppShell.tsx`, `ui/goose2/src/features/chat/hooks/__tests__/useChatSessionController.test.ts`, plus session-store / chat-controller / model-preference helpers

## Summary

Fix a perceived-latency bug in goose2 where opening a new tab inside a project sat blank for hundreds of milliseconds while async setup chained through `resolveSupportedSessionModelPreference → resolveSessionCwd → sessionStore.createSession → ACP prepareSession`. Splits the project-session path off the global async setup chain: creates a *local* session row optimistically (no awaited ACP call), routes the user to the chat view immediately, and binds the ACP session asynchronously in the background via the new `sessionStore.prepareSessionBinding` helper. Non-project (home) sessions retain the prior async path.

## Specific code references

- `AppShell.tsx:301-321`: new optimistic-project-session branch. `if (project) { ... }` constructs the model preference *synchronously* via `resolveSessionModelPreference({providerId, preferredModel})` (new sync helper at `sessionModelPreference.ts`) + `sanitizeSessionModelPreference(modelPreference, providerInventoryEntries.get(modelPreference.providerId))` rather than awaiting the prior `resolveSupportedSessionModelPreference` async call. Then `sessionStore.createLocalSession(...)` (new sync method) returns immediately, and `setActiveView("chat")` + `chatStore.setActiveSession(session.id)` swap the user into the chat view in the same tick. The `perfLog` line at `:316-318` instruments the win — `created local project session in Xms` should now report sub-frame numbers vs the prior multi-hundred-ms.
- `AppShell.tsx:323-335`: home-session (non-project) path is preserved verbatim with the prior async `resolveSupportedSessionModelPreference` + `resolveSessionCwd(null)` + `sessionStore.createSession`. This is the load-bearing scope discipline — only the project path is changed; users opening a non-project tab see no behavioral difference. Risk reduction is significant.
- `AppShell.tsx:494-501`: `acpPrepareSession(...)` direct call replaced with `sessionStore.prepareSessionBinding({...})`. The new wrapper accepts `personaId` and `projectId` as named fields rather than the prior options-object-with-personaId-only shape. The wrapper presumably encapsulates the "if local session, do binding; if already bound, no-op" logic that used to live inline.
- `useChatSessionController.test.ts:225+` (new test "binds an optimistic project session before setting its model"): the load-bearing regression-pin for the optimistic-session lifecycle. Test sequence is precisely: (a) state has a local session with `id: "local-session-1"` and *no* `acpSessionId`, (b) handleModelChange("gpt-4.1") is fired, (c) assertion that `mockAcpPrepareSession` was called with `("local-session-1", "openai", "/tmp/project", {personaId, projectId, knownNew: true})`, (d) post-await assertion the session now has `acpSessionId: "goose-session-1"` (returned from the mocked `acpPrepareSession`) and the model fields populated, (e) assertion `mockAcpSetModel` was finally called with the local session id. The `knownNew: true` option in (c) at `:307` is the load-bearing optimization signal that lets the binding code skip an "is this session already known?" round-trip.
- `useChatSessionController.test.ts:99-107` (additions in mock): new `if (modelId === "gpt-4.1") { onModelSelected?.({...providerId: "openai"}) }` arm in the `useAgentModelPickerState` mock. Symmetric with the existing `claude-sonnet-4` arm — needed to drive the new test, structurally consistent.
- `useChatSessionController.test.ts:171`: new `acpSessionId: "session-1"` field on the existing `session-1` fixture. This is required because the controller now distinguishes "session has a binding" (acp id present) from "session is local-only" (acp id absent), and the prior tests would have started failing without this disambiguation.
- `useChatSessionController.test.ts:240-247`: project fixture with `preferredProvider: "openai"`, `preferredModel: null` and a `workingDirs: ["/tmp/project"]`. The `preferredModel: null` arm is precisely the case the new `sanitizeSessionModelPreference` helper has to handle gracefully — falling back to provider's default model rather than failing.

## Reasoning

The fix is architecturally correct and well-tested. The optimistic-create-then-bind pattern is the standard answer to "async setup makes UI feel slow" — render synchronously with the best information available, then reconcile when the async result arrives. Three things the PR does right:

1. **Scope discipline.** Only the project-session path is changed; the home-session path is preserved. This minimizes regression surface for the 50%+ case.
2. **Dedicated `prepareSessionBinding` indirection.** Rather than inlining the "create local then bind" choreography across multiple call sites, the new method centralizes the lifecycle at the `sessionStore` boundary. The `knownNew: true` flag is a clean way to communicate the optimistic-creation context to the binding code.
3. **Regression test pinning the lifecycle order.** The "binds an optimistic project session before setting its model" test pins the exact sequence (`createLocalSession → prepareSessionBinding(knownNew:true) → setModel`) so a future refactor that changes the order (e.g., binding before model-set, or model-set before binding-completion) gets a loud failure rather than a subtle UX regression.

Concerns:
- **Failure mode for `acpPrepareSession` rejection.** If the background `prepareSessionBinding` call fails (network, ACP server crash, etc.), the user is already in the chat view with a "live-looking" session that has no ACP backing. The PR comment in this diff slice doesn't show the rejection-handling path. Verify there's a state machine that surfaces a "binding failed, tap to retry" UI, or at minimum that send-message attempts during the unbind window queue rather than silently fail.
- **Working-dir resolution moved out of the project path.** The prior code called `resolveSessionCwd(project)` synchronously in the create flow; the new code only calls `resolveSessionCwd(null)` in the home-session arm. The project-arm `prepareSessionBinding` call must therefore resolve the working directory itself (presumably from `project.workingDirs[0]` or similar). Worth confirming in the helper that the working-dir resolution still respects worktree creation if `useWorktrees: true` is set on the project.
- **Test coverage doesn't include the binding-failure path.** The new test mocks `acpPrepareSession.mockResolvedValueOnce("goose-session-1")` (success). A symmetric `mockRejectedValueOnce(...)` test arm pinning the error-state UX would close the regression-discipline circle.

## Verdict

**merge-after-nits**
