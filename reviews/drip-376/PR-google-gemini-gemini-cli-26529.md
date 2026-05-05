# google-gemini/gemini-cli #26529 — feat(agent): formalize first-class tool lifecycle states and status mapping

- Author: mbleigh
- Head SHA: `4a4f54c20d0d7183de8498f2fc603983ba3d70c3`

## Files changed (top)

- `packages/core/src/agent/types.ts` (+14 / -0) — adds `ToolEventStatus` union
- `packages/core/src/agent/legacy-agent-session.ts` (+83 / -0) — message-bus subscription, status threading
- `packages/cli/src/ui/hooks/useAgentStream.ts` (+16 / -8) — refactors status derivation
- `packages/core/src/agent/event-translator.ts` (+2 / -0)
- `scripts/build_package.js` (+9 / -2) — unrelated rmSync-before-cpSync fix

## Observations

- `packages/core/src/agent/types.ts:230-236` — new `ToolEventStatus = 'pending' | 'pending_input' | 'executing' | 'succeeded' | 'errored' | 'aborted'` is a clean six-variant union covering the full tool lifecycle including the previously-implicit `pending_input` (awaiting approval) and `aborted` (cancelled) states. This is the load-bearing modeling change.
- `packages/core/src/agent/types.ts:244, 268, 291` — `status: ToolEventStatus` added as a **required** field on `ToolRequest`, `ToolUpdate`, and `ToolResponse`. This is a breaking change to the event protocol: any external consumer constructing these events by hand will now fail typecheck. For an internal-only protocol this is fine; for any plugin/extension API surface, a migration note is warranted.
- `packages/cli/src/ui/hooks/useAgentStream.ts:227-238` — status derivation moves from reading `event._meta?.legacyState?.status` (which had only 3 values: `executing | error | success`) to reading `event.status` directly with five mapped variants. The new `pending_input → AwaitingApproval` and `aborted → Cancelled` mappings are net-new UX states the CLI will now correctly render. Good.
- `packages/cli/src/ui/hooks/useAgentStream.ts:278-286` — second status block in the tool-response branch defaults `status = CoreToolCallStatus.Success` then overrides on `errored | aborted | succeeded`. Subtle issue: if `event.status` is `'pending'` or `'executing'` or `'pending_input'` (which shouldn't occur on a tool-response event but is type-allowed), the default of `Success` is *wrong*. Consider an explicit assertion-or-warn for the unexpected branches, or narrow the type at this site.
- `packages/core/src/agent/legacy-agent-session.ts:101-105` — `messageBus.subscribe(MessageBusType.TOOL_CALLS_UPDATE, this._handleToolCallsUpdate.bind(this))` registers a subscription with no corresponding unsubscribe in any disposal/teardown path visible in the diff. If `LegacyAgentProtocol` instances are short-lived per session, this leaks one subscriber per session for the lifetime of the message bus. Confirm the bus has a sweep mechanism or add an unsubscribe in the dispose path.
- `packages/core/src/agent/legacy-agent-session.ts:290-294` — `status: response.error ? 'errored' : tc.status === 'cancelled' ? 'aborted' : 'succeeded'` — correct three-way branching. Good.
- `packages/core/src/agent/event-translator.ts:238, 261` — translator now emits `status: 'pending'` on tool_request and `status: event.value.error ? 'errored' : 'succeeded'` on tool_response. Consistent with the new contract.
- `scripts/build_package.js:51-58` — adding `rmSync` before `cpSync` is unrelated to the type-system work but is a real fix for the docs-copy step that previously failed on rebuilds when the target existed. Reasonable to bundle but a separate commit would have been cleaner.

## Verdict

`merge-after-nits`

Strong, well-modeled refactor that promotes implicit lifecycle states to first-class type-system citizens. Two nits worth resolving pre-merge: (1) the `useAgentStream.ts:278-286` default-to-Success-then-override pattern silently misclassifies the unreachable-but-type-valid `pending`/`executing`/`pending_input` branches — narrow or warn, and (2) the `legacy-agent-session.ts:101-105` message-bus subscription needs a teardown counterpart to prevent per-session leaks. The bundled `build_package.js` fix is fine but should ideally be a separate commit.
