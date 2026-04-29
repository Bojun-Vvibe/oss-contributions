---
pr: sst/opencode#24984
sha: 62b7694465283e95adb556b60acbd6ac004dcf23
verdict: merge-after-nits
reviewed_at: 2026-04-29T18:31:00Z
---

# fix(core): reconnect editor context for session directory

## Context

Before this PR, `EditorContextProvider` in `packages/opencode/src/cli/cmd/tui/context/editor.ts` resolved
the editor lock file once at process startup using `process.cwd()`. After a
session switch, opencode kept talking to whatever editor server was attached
when the binary first started — wrong file tree, wrong selection ranges, sometimes
a dead socket. The author moves the connection state out of `onMount` into
provider-scoped `let` bindings and adds an explicit `directory` variable so
the provider can re-resolve `resolveEditorConnection(directory)` whenever
the active session changes.

## What's good

- The lift from `onMount` closure to provider-scope (`let socket`,
  `let directory = process.cwd()`, etc.) is the right shape — re-mounting
  was never the trigger; *session change* is. The reconnect call site
  ends up colocated with `setStore("server", …)`, which is the only place
  state actually needs to be invalidated.
- `WebSocketImpl` injection (`init: (props: { WebSocketImpl?: typeof WebSocket })`)
  is the testability hook the existing test suite needed — the new
  `editor-context.test.tsx` (referenced in the PR body) can stub the socket
  without monkey-patching the global. Good restraint: it falls back to the
  real `WebSocket` at the call site rather than at module load.
- Same-directory short-circuit (avoid socket churn when nothing changed)
  is mentioned in the PR body and is the right safety: editor servers
  often have warm-up costs, and an unconditional reconnect on every
  session activation would be a regression.

## Concerns / nits

1. The `socket.readyState !== 1` literal in the new `send()` replaces
   `WebSocket.OPEN`. That works but loses self-documentation and risks
   diverging if `WebSocketImpl` is a polyfill that uses a different enum.
   Prefer `WebSocketImpl.OPEN` (or pull it onto a local `const OPEN = WebSocketImpl.OPEN`).
2. `lastSubmittedEditorSelectionKey` in `prompt/index.tsx` is a closure-scoped
   `let` inside `Prompt(props)`. If `<Prompt>` ever remounts (route change,
   keyed re-render), the dedupe state resets and the same selection will be
   re-submitted as a fresh part. Worth a comment or a `createSignal` so
   intent is explicit.
3. The PR drops `editor.clearSelection()` after submit and replaces it with
   the dedupe-key assignment. That changes semantics: the editor selection
   stays "live" across submits. Confirm with the design owner that this is
   intentional — for users who want each prompt to start clean, this is a
   subtle UX shift.

## Verdict

`merge-after-nits` — fix the `WebSocket.OPEN` literal, add a one-line
comment explaining the `lastSubmittedEditorSelectionKey` lifecycle, and
double-check the `clearSelection` removal is intentional.
