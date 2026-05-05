# QwenLM/qwen-code #3842 — feat(core): add signal.reason convention for ShellExecutionService (#3831 PR-1 of 3)

- **Head SHA:** `6cbab376d7ab8f1ffc554f545c9ca955ae8d6610`
- **Author:** @wenshao
- **Verdict:** `merge-as-is`
- **Files:** +209 / −1 across `packages/core/src/services/shellExecutionService.{ts,test.ts}`

## Rationale

This is **PR-1 of a 3-PR sequence** (per #3831) implementing Phase D part (b) — Ctrl+B promote of a running foreground shell to a background-managed shell. PR-1 is **pure plumbing with zero behavior change** for any existing caller, because no caller in this PR sets the new `signal.reason` to anything other than its current default. That scoping discipline is the headline reason this is `merge-as-is`: the foundation is independently mergeable and revertable, and the design issue (#3831) is still open for the caller-side semantics that PR-2/PR-3 will land.

The contract is defined at `shellExecutionService.ts:51-54`:

```ts
export type ShellAbortReason =
  | { kind: 'cancel' }
  | { kind: 'background'; shellId?: string };
```

with the discrimination logic added in **both** abort handlers — `executeWithPty` at line 957-979 and `childProcessFallback` at line 546-573 — sharing the same shape: read `abortSignal.reason as ShellAbortReason | undefined`, branch on `reason?.kind === 'background'`, skip the kill, drop the child from the `activePtys` / `activeChildProcesses` set (so `ShellExecutionService.cleanup()` won't kill it later either), flush a snapshot of captured output, and resolve the result Promise *immediately* with `promoted: true`. The caller has accepted ownership of the live child by this point.

Three correctness details deserve callouts. (1) The `activeChildProcesses.delete(child.pid)` / `activePtys.delete(ptyProcess.pid)` is critical — without it, the process-exit handler that `ShellExecutionService.cleanup()` registers would later kill the promoted child on Ctrl+C or process shutdown. (2) The PTY snapshot uses `serializeTerminalToText(headlessTerminal) ?? ''` (line 970) inside a `try/catch` — exactly the right defensive shape, since terminal serialization can race mid-render and return undefined in test mocks; the empty fallback is acceptable because the rawOutput buffer is also returned. (3) Idempotency on real exit is preserved: when the promoted child eventually exits naturally, its existing `child.on('exit')` / `ptyProcess.onExit` handler still fires, but `cleanup()` and `resolve()` are both idempotent (`exited` flag check at line 962 for PTY, the closure-captured `resolve` for child_process). The exit handler runs to completion as wasted no-op work, but no leak.

The discriminated-union shape (vs. an enum or a separate `promotionMode` parameter) is the right call for the rare-path semantics. Per the PR body, three alternatives were considered: (a) new `execute()` parameter — adds always-present API surface for a <1%-of-the-time path; (b) pre-allocate hidden `BackgroundShellEntry` for every foreground spawn — touches `ShellExecutionService` heavily and wastes memory in the common case; (c) `signal.reason` as takeover channel (this PR) — reuses `AbortController` plumbing every caller already has, 5-line addition to the existing abort handler, zero overhead in the common case, leverages Node's native `AbortSignal.reason` (Node 17.2+, covered by the `engines: >=20`). Option (c) is correct.

Test coverage at `shellExecutionService.test.ts` adds 4 new cases (54/54 total pass per PR body): two on the PTY path (line 614-665) and two on the child_process fallback path (line 1198-1257). Each path tests both directions of the discrimination — `{ kind: 'cancel' }` still tree-kills via `process.kill(-pid, 'SIGTERM')` (pinning that the historical default is *not* mistakenly routed through the background branch), and `{ kind: 'background' }` skips kill (assert `mockProcessKill not toHaveBeenCalledWith` SIGTERM/SIGKILL on the group pid, `mockPtyProcess.kill not toHaveBeenCalled`, `result.promoted === true`, `result.exitCode === null`, `result.signal === null`). The child_process case additionally pins the snapshot semantic by emitting `'line1\nline2\n'` to stdout pre-abort and asserting `result.output` contains both lines — exactly what PR-2 will need to seed the `BackgroundShellEntry`'s on-disk output file.

The caller contract is also documented inline at `shellExecutionService.ts:30-46`: "Callers MUST attach their own listeners (data / exit / error) to the live child *before* calling `abortController.abort({ kind: 'background', ... })`" — that's the right invariant to encode in a doc comment now, while PR-2's caller is still being designed against #3831. The `ShellExecutionResult.promoted?: boolean` field (line 71-78) is optional, so existing consumers compile against the new shape without source changes.

This is exemplary PR-sequencing: the foundation is contract-first (discriminated union + doc comment), the behavior change is zero for existing paths (pinned by the cancel-path tests), and the new behavior is exercised by tests that don't depend on PR-2's caller existing yet. Merge as-is.