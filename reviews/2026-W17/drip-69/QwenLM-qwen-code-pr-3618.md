# QwenLM/qwen-code #3618 — fix(vscode-companion): fill slash commands into input on Enter instead of auto-submitting

- **Author:** yiliang114
- **Head SHA:** `643d9829fd46a5903e3f1d2d146ad5a076b7ccdf`
- **Size:** +70 / -40 (8 files)
- **URL:** https://github.com/QwenLM/qwen-code/pull/3618

## Summary

Fixes #1990. The completion menu in the VSCode companion was auto-
submitting *any* command on Enter, including ones that take arguments
(skills, custom commands). That made arg-bearing commands unusable from
the menu — they sent before the user could type. The PR introduces an
ACP-protocol-level signal (`AvailableCommand.input`) carrying a hint
string when a command accepts arguments, then uses it on the webview
side to decide auto-submit vs fill-into-input.

## Specific findings

- **Protocol change is the right shape.**
  `packages/channels/base/src/AcpBridge.ts:23-27` adds an optional
  `input?: { hint: string } | null` field to `AvailableCommand`. The
  optional + nullable type is a small ACP-wire ambiguity (clients have
  to handle both `undefined` and `null`), but the project's existing
  use elsewhere is consistent with this convention and the populator at
  `Session.ts:980-993` always emits one of the two forms explicitly:

  ```ts
  input:
    cmd.kind !== CommandKind.BUILT_IN || cmd.completion != null
      ? { hint: cmd.argumentHint ?? '' }
      : null,
  ```

  The decision rule — *"non-built-in OR has a completion function"* —
  is the right denotation for "accepts arguments" given the codebase's
  existing categorisation. Skills are non-built-in, custom commands are
  non-built-in, anything that opted into a `completion` callback
  effectively declared "I take input." The empty-string fallback for
  `argumentHint` is correct: callers receive `{ hint: "" }` rather than
  `{ hint: undefined }`, so downstream rendering doesn't have to guard
  for `undefined`.

- **Webview behaviour split is clear.**
  `packages/vscode-ide-companion/src/webview/App.tsx:693+` (visible
  partial diff) refactors the per-command if-ladder for client-side
  commands (`/auth`, `/account`) into a `clientActions` map. That's a
  good refactor regardless of this PR's scope — but it does mean the
  diff is slightly larger than strictly needed for the fix, and a
  reviewer should make sure the new map is exhaustive for the previous
  if-ladder cases.

- **The Tab-vs-Enter contract is preserved.** Per the PR body, Tab
  always fills (unchanged) and Enter newly distinguishes by the `input`
  field. That's the right UX: Tab has always meant "complete," Enter
  has always meant "go," and "go" for an arg-bearing command should
  reasonably mean "give me a chance to type the args first."

- **`/login` → `/auth` migration is bundled.** Per the PR body and the
  diff snippets in `useMessageSubmit.ts` and `WebViewProvider.ts`, the
  alias and a stale comment update are folded in. Fine to ship together
  but should be called out in the changelog so users with `/login`
  muscle memory aren't surprised by the rerouting (legacy alias is
  preserved per the PR body, so no functional break).

- **`useWebViewMessages.ts` adds an `authCancelled` handler** to clear
  the waiting state when the auth picker is dismissed. Sensible
  defensive change (otherwise dismissing leaves the UI stuck). Tested
  per `useWebViewMessages.test.tsx:218`.

- **Reviewer Test Plan TODO.** PR body marks "Add before/after
  recording" as TODO. For a UI behaviour change that touches the
  primary command-entry path, a recording (or even a textual
  before/after) would meaningfully accelerate review.

## Verdict

`merge-after-nits`

(Add the before/after demo per the author's own TODO, and call out the
`/login`→`/auth` reroute in the user-facing changelog.)
