# Review: sst/opencode #25439 — fix(skill): /<skill-name> now calls the skill tool directly

- **PR**: https://github.com/anomalyco/opencode/pull/25439
- **Head SHA**: `c442aac61d94c3d8d31670c63cb7aa2c592e72c5`
- **Diff size**: ~580 lines across submit handling, prompt component, and `session/prompt.ts`

## What it changes

Two layered changes:

1. **Slash-command parsing rewrite** in `packages/app/src/components/prompt-input/submit.ts:74-75`
   and `packages/opencode/src/cli/cmd/tui/component/prompt/index.tsx:830-849`. Replaces the
   ad-hoc `text.split(" ")` / `firstLine.split(" ")[0].slice(1)` with a single regex
   `^\/([^\s]+)([\s\S]*)$`. Multi-line args and trailing-space arg handling collapse into
   one well-defined matcher.
2. **Direct skill invocation path** in `packages/opencode/src/session/prompt.ts:1516-1741`.
   New `activateCommandSkill` Effect manufactures a synthetic user message + assistant
   message + running tool part for the `skill` tool, runs it via `skillTool.execute`, and
   feeds the output back through the standard tool-state lifecycle (`updatePart` with
   `running` → `completed`/`error`). The existing `command()` function then dispatches to
   either `activateCommandSkill` (when `cmd.source === "skill"`) or the prior prompt path.

## Assessment

The regex change at submit.ts:74 is a clean improvement. Old code:

```ts
const [head, ...tail] = text.split(" ")
const cmd = head?.startsWith("/") ? head.slice(1) : undefined
```

silently dropped multi-line semantics — `tail.join(" ")` collapses any embedded newlines
into spaces. New code preserves them via `[\s\S]*`. Same fix in the TUI prompt at
index.tsx:830-849.

Concerns about `activateCommandSkill`:

- **Cost accounting**: line 110 hardcodes `cost: 0` on the assistant message and never
  updates it. Skill tool execution may call providers underneath (most do); whatever cost
  those incur won't roll up into this synthetic assistant turn. If session totals are
  computed from message.cost, this leaks to zero. Worth a comment if intentional.
- **Token accounting**: same problem at line 113 — `tokens: { input: 0, output: 0, ... }`.
- **Error path message state**: line 192 unconditionally calls
  `sessions.updateMessage(assistantMessage)` with `time.completed = Date.now()` *before*
  branching on `Exit.isFailure`. Then on failure (line 197) the part is updated to
  `status: "error"` and the error is rethrown. The assistant message stays marked
  completed, which is fine, but the message itself contains zero indication that the only
  part errored. Downstream renderers had better key off part state, not message state.
- **`skillTool.id` lookup at line 79**: `(yield* registry.all()).find((item) => item.id === "skill")`
  could be `Option.fromNullable` + `Effect.fail` instead of `throw new Error(...)`. The
  rest of this file uses Effect-native error handling; throwing here breaks the pattern
  and skips the `bus.publish(Session.Event.Error, ...)` path that other failure sites use.
- **Followup args parsing** in `command()` at lines 274-279: `/^\s/.test(input.arguments) ?
  input.arguments : \`\${space}\${input.arguments}\`` — the conditional reads as "if args
  already start with whitespace, keep them, else prepend a space." That's correct, but the
  variable name `followupSuffix` plus the `${" "}\${input.arguments}` template is hard to
  scan. A two-line helper would help.

## Verdict

`request-changes` — the parsing rewrite is good but the synthetic-message construction has
two real concerns (zero cost/token accounting, throw-instead-of-Effect.fail at line 79)
and one nit (variable naming around line 274). I'd want at least the cost/token issue
addressed before merge, since it silently corrupts session-level accounting for any
session that uses a skill command.
