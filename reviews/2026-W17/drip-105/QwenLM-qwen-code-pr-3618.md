---
pr: https://github.com/QwenLM/qwen-code/pull/3618
sha: ffce3d17
diff: +167/-40
state: OPEN
---

## Summary

Reworks how the VS Code companion handles slash-command selection so that commands accepting arguments (skills, custom commands with completion) get filled into the input box for further typing on Enter, while built-in commands without completion still auto-submit immediately. The contract is moved from "decide on the client by name" to "decide based on the `input` field the server attached to each `AvailableCommand`", which requires a coordinated change in `Session.ts` (server side) plus the webview's selection handler and the ACP type.

## Specific observations

- `packages/channels/base/src/AcpBridge.ts:25-26` — adds `input?: { hint: string } | null` to `AvailableCommand`. Three-state typing (`{hint}` | `null` | `undefined`) is the right shape for "command needs input with a hint" vs "command takes no input, auto-submit" vs "legacy/unknown, fall back to old behavior". Make sure the protocol doc (if any) calls out that `null` is semantically distinct from `undefined`; client code at `App.tsx:824` checks `!serverCmd.input` which collapses both to "auto-submit", which works today but locks in the conflation.
- `packages/cli/src/acp-integration/session/Session.ts:984-991` — the server-side decision is:
  ```ts
  input:
    cmd.kind !== CommandKind.BUILT_IN || cmd.completion != null
      ? { hint: cmd.argumentHint ?? '' }
      : null,
  ```
  Reads as: "if it's not a built-in, OR it has completion, give it an input box; otherwise auto-submit." That correctly captures "skills (not built-in) → fill" and "built-in with completion → fill" while keeping "built-in without completion (e.g. `/clear`) → auto-submit". The `cmd.argumentHint ?? ''` fallback hands an empty-string hint when `argumentHint` is missing, which is the right neutral placeholder. Old code at the same location was `cmd.argumentHint ? { hint: cmd.argumentHint } : null` — i.e. used the *presence* of an argument hint as the input flag. New code uses kind+completion, which is correct because some commands accept input but don't ship a written hint.
- `packages/vscode-ide-companion/src/webview/App.tsx:822-836` — the selection handler now branches on `serverCmd.input`. The new control flow is: client-actions (auth/account/model) handled first with their own table; then server commands; if `!serverCmd.input && !fillOnly` auto-submit, else fall through to the generic insertion path that lets the user keep typing. The collapsing of three previously-separate `if (itemId === 'auth' / 'account' / 'model')` blocks into a `clientActions` lookup table at `:798-804` is a nice cleanup; `model` was previously falling through to one of the bespoke blocks below, so the table form makes the symmetry explicit.
- `Session.test.ts:241-244` — test updated to expect `input: null` on a command with no `argumentHint`. Matches new contract.
- `packages/cli/src/acp-integration/session/Session.ts:84` adds `import { CommandKind }` — the import path crosses a layering boundary (`acp-integration` reaching into `ui/commands/types.js`). That coupling already existed conceptually (the ACP layer translates UI commands), but a direct import makes it concrete. If the project has a dependency-direction lint, this may flag; otherwise it's fine but a candidate for extracting the kind constant into a shared types module.
- `packages/vscode-ide-companion/src/webview/hooks/useMessageSubmit.ts:101-115` — collapses `/login` handling into "match `/auth` OR `/login`" with the comment calling `/login` a "legacy alias". Good migration discipline — the alias keeps existing user muscle memory working while the canonical name becomes `/auth`.
- The webview-companion changes are reactive UX and hard to fully validate from diff alone; the `App.test.tsx +56` additions cover the new `commitCommandItem` and `clearCommandItem` cases but I'd want to also see a test that exercises the *fall-through* path: select a command with `input: { hint }`, hit Enter, assert the input box now contains `/<command>` *and* is not submitted.

## Verdict

`merge-after-nits` — the contract migration (server decides per-command via the `input` field) is the right architecture, both the server and client sides implement it consistently, and the legacy-alias treatment of `/login` is thoughtful. Two small follow-ups: (1) clarify (test or comment) the semantic distinction between `input: null` and `input: undefined`, since `App.tsx`'s `!serverCmd.input` collapses them, (2) add a webview test that asserts the fill-then-don't-submit path for an input-accepting command at Enter (not just Tab).

## What I learned

When a client previously hardcoded "which commands take args" by name, the right migration is to push the decision to the producer (server) and ship a structured field, not a flag. Three-state typing (`{...}` | `null` | `undefined`) is a useful trick when you need to distinguish "no input needed" from "decision deferred / legacy" — but it only works if every consumer treats the three states distinctly, otherwise the type is more honest than the code.
