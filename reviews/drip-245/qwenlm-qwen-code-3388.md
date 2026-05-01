# PR #3388 — feat(hooks): add prompt hook type with LLM evaluation support

- Repo: QwenLM/qwen-code
- Head: (unrecorded in list snapshot — large feature PR)
- URL: https://github.com/QwenLM/qwen-code/pull/3388
- Verdict: **needs-discussion**

## What lands

Adds a fourth hook executor type — `prompt` — alongside the existing
`command`, `http`, and `function` types. Prompt hooks send the hook input
JSON to an LLM via a user-supplied prompt template (with `$ARGUMENTS`
placeholder), and parse a `{ok: bool, reason?, additionalContext?}` JSON
response to decide whether to allow or block the operation. +1676 / -39
across docs, settings schema, runner, UI, and tests.

## Specific findings

- `packages/cli/src/config/settingsSchema.ts:140` extends the hook type
  enum from `['command', 'http']` to `['command', 'http', 'prompt']` and
  adds two new schema fields: `prompt` (required template, with
  `$ARGUMENTS` placeholder docs) and `model` (optional model override).
  Schema-level validation will reject unknown types, so this is a real
  contract extension.
- `packages/core/src/hooks/hookEventHandler.ts:707-708` widens
  `getHookTypeFromResult` return type from
  `'command' | 'http' | 'function'` to `'command' | 'http' | 'function'
  | 'prompt'` — correct, but downstream consumers (telemetry, logging,
  any switch on hook type) need to handle the new variant exhaustively.
  Worth searching for switch/match statements over hook type to ensure
  none silently fall through.
- `HooksManagementDialog.tsx:55-57` adds the validation arm:
  ```
  if (obj['type'] === 'prompt') {
    return 'prompt' in obj && typeof obj['prompt'] === 'string';
  }
  ```
  Correct minimal validation but doesn't check for empty `prompt`
  strings. An empty prompt template would be a footgun (the LLM gets
  literally nothing) — should reject `prompt.trim().length === 0`.
- UI handling at `HookConfigDetailStep.tsx:62-68` and `HookDetailStep.tsx:104-113`
  dispatches the right way (separate render branches per type, prompt
  truncated to 50 chars in the list view). Looks fine.
- Documentation at `docs/users/features/hooks.md:33-227` is thorough,
  with two worked examples (`Stop` and `PreToolUse`) showing realistic
  prompt templates.

## Why "needs-discussion" rather than nits

- **Cost / trust model not addressed.** A `PreToolUse` prompt hook
  fires an LLM call before *every* tool invocation. On a tool-heavy
  agent loop (10+ tool calls per turn) the user is paying for double
  the model traffic. Worse: if the user's main loop is on `qwen-max`
  but the prompt hook defaults to the same model (the schema docs say
  "defaults to your current model"), they're getting `qwen-max`-priced
  evaluations on every gate. The default should probably be a cheap
  fast model (the docs reference `fastModel` in one place), and the
  cost contract should be explicit.
- **Fail-open as the documented contract is risky.** The PR description
  says "Fail-open: invalid LLM responses default to allow." For a
  *security* hook (the second example explicitly markets prompt hooks as
  evaluating "Dangerous commands (rm -rf, curl | sh, etc.)"), fail-open
  is exactly the wrong default — a malformed LLM response from a
  dropped connection or rate-limit error becomes "tool call allowed."
  This needs to be at minimum *configurable* (`failureMode: 'allow' |
  'block'`) and the doc example for the security hook should call it
  out loudly.
- **Prompt-injection surface.** The hook input is interpolated into the
  prompt template via `$ARGUMENTS`. For a `PreToolUse` hook that
  receives raw tool args, a malicious prompt in user input that
  reaches a tool call could craft a `$ARGUMENTS` payload that
  flips the gate to `{"ok": true}`. There's no mention of escaping or
  sandboxing the interpolation. At minimum the docs should warn users.
- **`HookType.Prompt` enum addition** isn't visible in the diff window
  I read — needs a confirmation that all `match`/`switch` over
  `HookType` got an arm added (TypeScript will catch most cases via
  exhaustiveness, but loose `string` typing in older code may not).

## Verdict rationale: needs-discussion

The mechanics work and the test coverage looks reasonable, but the
security-and-cost framing of the feature deserves an explicit design
discussion before merge. The wenshao "Changes requested" review status
on the PR aligns with this read. Reasonable forward path: (1) make
default model `fastModel` (cheaper), (2) make failure mode configurable
with `block` as the recommended default for security hooks, (3) add a
prompt-injection warning paragraph to the docs.
