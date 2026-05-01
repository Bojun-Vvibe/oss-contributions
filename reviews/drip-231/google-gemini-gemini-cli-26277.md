# google-gemini/gemini-cli#26277 — docs(sdk): add JSDoc to all exported interfaces and types

- **Author:** fauzan171
- **Head SHA:** `22001650ede2ec1ded343d62e0c34ec88a8146dc`
- **Base:** `main`
- **Size:** +343 / -0 across 7 files (pure docs additions, no code change)
- **Files changed:** `packages/sdk/src/{agent,fs,session,shell,skills,tool,types}.ts`

## Summary

Pure documentation PR adding JSDoc comments to every exported interface, type, class, function, and method in the public SDK surface (`@google/gemini-cli-sdk` package). No runtime behavior change. Closes a real DX gap where IDE hover-tooltips on the SDK's public API showed only the type signature with no description, leaving SDK consumers reading the source to understand what `sendStream`, `tool()`, `AgentFilesystem`, `ModelVisibleError`, etc. actually do.

## Specific code references

- `packages/sdk/src/agent.ts:9-29`: `GeminiCliAgent` class JSDoc includes a complete `@example` block showing the canonical end-to-end flow — `new GeminiCliAgent({ instructions, tools })` → `agent.session()` → `await session.initialize()` → `for await (const event of session.sendStream(...))`. This is the highest-value addition because the class is the SDK's main entry point and a working example in the IDE tooltip is significantly faster than reading the README.
- `agent.ts:37-46`: the `session(options?)` JSDoc correctly documents the load-bearing contract that `options.sessionId` is *optional* and a new ID is auto-generated otherwise, with the `{@link GeminiCliSession}` cross-link rendering as a clickable navigation in IDEs that support TSDoc.
- `agent.ts:49-58`: `resumeSession(sessionId)` documents the `@throws {Error}` for missing sessions — important for callers to know they need to handle this exception rather than getting a runtime surprise. The "looks up the session's conversation history from storage and replays it" sentence describes the actual semantics rather than just restating the function name.
- `session.ts:104-117`: `initialize()` documents the load-bearing **idempotent** contract ("calling it multiple times has no effect after the first successful initialization"). This is exactly the kind of behavioral promise that has no type-system representation and must live in JSDoc.
- `session.ts:125-145`: `sendStream` documents the full agentic loop semantics ("sends the user prompt, streams model responses, executes any tool calls the model requests, and continues the loop until the model produces a final response with no tool calls") — this is critical because the loop semantics are not obvious from the `AsyncGenerator` return type alone. The `@example` block demonstrating the `event.type === GeminiEventType.ModelResponse` discriminated-union pattern is the right way to teach the consumer the event shape.
- `tool.ts:209-237`: `ToolDefinition<T>` JSDoc documents each field including the load-bearing `sendErrorsToModel?: boolean` default of `false` — without this, callers would have to either read the source or guess.
- `tool.ts:240-260`: the `Tool<T>` interface JSDoc clarifies the action signature contract — params come pre-validated from the Zod schema, context is optional, return value is "serialized (to JSON if not already a string) and sent to the model". The "if not already a string" detail is non-obvious behavior worth documenting.
- `tool.ts:281-305`: the `tool()` factory function `@example` is correct and complete — the `import { z, tool } from '@google/gemini-cli-sdk'` line, the inline Zod schema, and the async action returning a string. This works as copy-paste-and-modify boilerplate.
- `shell.ts:157-168`: the `@remarks` block on `SdkAgentShell` documents the load-bearing implementation detail that "stderr is combined into stdout by the underlying ShellExecutionService. As a result, the stderr field of the returned AgentShellResult will be empty, and both output and stdout will contain the combined output." This is exactly the kind of platform-specific quirk a consumer would otherwise discover only through debugging — it deserves to be in the IDE tooltip.
- `types.ts:317-326,330-378`: every field of `GeminiCliAgentOptions` documented including `tools`, `skills`, `model`, `cwd`, `debug`, `recordResponses`, `fakeResponses`. The `recordResponses` / `fakeResponses` pair is documented as record-for-replay-debugging-or-testing, which matches the testing usage in the SDK examples.
- `types.ts:380-401`: `AgentFilesystem.readFile` documents the load-bearing **null-on-deny** contract ("returns the file contents as a UTF-8 string, or `null` if the file does not exist or access is denied") — this differs from `writeFile` which throws, and the asymmetry is exactly what JSDoc must explain because the type signature alone doesn't.

## Verdict

**merge-as-is**

Pure additive docs change with zero code modification, +343 / -0 line accounting, and every JSDoc comment is substantive (describes contract, not just restates the type signature). The `@example` blocks on `GeminiCliAgent`, `tool()`, and `sendStream` are copy-paste-runnable. The `@remarks` block on `SdkAgentShell` documents a real platform quirk (stderr-into-stdout combining) that a consumer would otherwise hit through debugging. The `{@link ...}` cross-links between `GeminiCliAgent` ↔ `GeminiCliSession` ↔ `AgentFilesystem` ↔ `Tool` ↔ `ToolDefinition` form a navigable graph in TSDoc-aware IDEs. Documents the load-bearing **idempotent** contract on `initialize()`, the **null-on-deny** contract on `readFile`, the **throws-on-deny** contract on `writeFile`, and the **sendErrorsToModel default false** contract on `ToolDefinition` — all behavioral promises that previously lived only in source. Net win for SDK adoption with no risk surface. Merge.
