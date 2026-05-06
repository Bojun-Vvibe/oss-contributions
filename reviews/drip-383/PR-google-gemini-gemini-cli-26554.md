# google-gemini/gemini-cli#26554 — fix(acp): move tool explanation from thought stream to tool call content

- **Head SHA**: `71e7b29d0ca1f4150baece61b8668310ba83adc1`
- **Author**: sripasg (Sri Pasumarthi)
- **Stats**: +135 / -7, 2 files (related: #1659)

## Summary

ACP UX fix. Previously the tool-invocation explanation (`invocation.getExplanation()`) was emitted as a separate `agent_thought_chunk` ACP update, which created noise in chat clients (JetBrains/Xcode) by interleaving raw JSON-stringified tool arguments with the agent's prose stream. PR moves the explanation into the `content` array of the structured `tool_call` / `RequestPermissionRequest` payload so it stays visible in the IDE's tool-call panel without polluting the chat thought stream.

## Specific citations

- `packages/cli/src/acp/acpSession.ts:619-624` (deletion): the entire `if (explanation) { await this.sendUpdate({ sessionUpdate: 'agent_thought_chunk', content: { type: 'text', text: explanation } }); }` block is removed.
- `acpSession.ts:644-649` (addition, permission path): when `content.length === 0 && explanation`, push `{ type: 'content', content: { type: 'text', text: explanation } }` into the permission-request `toolCall.content` array. The `length === 0` guard correctly defers to existing diff content (the regression at #1572 referenced in PR body).
- `acpSession.ts:711-716` (addition, no-permission path): unconditionally push the explanation content into the `tool_call` update's `content` array — no length guard here because this branch builds `content` fresh.
- Test at `acpSession.test.ts:589-651` (`should add explanation to tool call content instead of thought chunk`) asserts both negative (`sessionUpdate` was *not* called with `agent_thought_chunk`) and positive (`requestPermission` was called with `toolCall.content` containing the explanation).
- Test at `acpSession.test.ts:653-708` (`should add explanation to tool_call update content instead of thought chunk when no permission required`) covers the no-permission branch.

## Verdict

**merge-after-nits**

## Rationale

Real UX fix with thorough test coverage of both code paths and asymmetric (negative+positive) assertions. The asymmetric guarding is mildly suspicious: permission path checks `content.length === 0` to avoid clobbering diff content, but the no-permission path at `:711-716` doesn't — if any future change makes that branch non-empty before this push, the explanation would duplicate. A defensive `if (content.length === 0 && explanation)` would harmonize both branches even though the current code is reachable-correct. Also, the validation matrix (Linux/Windows + Docker/Podman/Seatbelt) is mostly unchecked — only macOS+npm-run is verified — but this is a pure ACP-protocol change with no platform-specific code, so test coverage compensates. Confirm no downstream IDE consumer (Xcode/JetBrains plugin) was relying on the `agent_thought_chunk` shape for tool explanations specifically; a CHANGELOG note would help.

