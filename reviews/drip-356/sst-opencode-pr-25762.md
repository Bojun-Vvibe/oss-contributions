# sst/opencode#25762 — fix: prevent shell commands from killing all Node.js processes

- Head SHA: `4c7cf5639030b394337f21ded6afecea7c84ce3d`
- Author: Xelson431
- Verdict: **request-changes**

## Summary

Adds three layers of defense to stop the agent from killing its own
host process: a system-prompt warning, an inline tool-prompt warning,
and a regex denylist that throws before `Shell.ps()` is invoked.
Targets `taskkill /F /IM node.exe`, `killall node`, `pkill node`, and
`Get-Process node | Stop-Process`.

## Specific references

- `packages/opencode/src/tool/shell.ts:32-43` — the denylist:
  ```ts
  const DANGEROUS_COMMAND_PATTERNS = [
    /taskkill\s+.*\/?[Ff]\s+.*\/?[Ii][Mm]\s+node\.?exe/i,
    /taskkill\s+.*\/?[Ii][Mm]\s+node\.?exe/i,
    /killall\s+node/i,
    /pkill\s+node/i,
    /Get-Process\s+.*node\s*\|\s*Stop-Process/i,
  ]
  ```
- `packages/opencode/src/tool/shell.ts:617` — the throw site, gated on
  `isDangerousCommand(params.command)`. Throws synchronously before any
  shell process is spawned.
- `packages/opencode/src/session/system.ts:62-67` and
  `packages/opencode/src/tool/shell/shell.txt:11-15` — duplicated copy
  of the same warning paragraph in both the session system prompt and
  the per-tool prompt.

## Reasoning

The motivation is sound — the agent has demonstrably crashed itself by
issuing `killall node` — but the implementation has structural problems
that should be addressed before merging.

**The regex layer is trivially bypassable.** `kill -9 $(pgrep node)`,
`for p in $(pidof node); do kill $p; done`, `taskkill /F /PID <pid>`
where `<pid>` happens to be the agent itself, `killall -9 node` with
weird whitespace, or any PowerShell `Stop-Process -Name 'node'`
variant all sail right through. A model that's been told "don't kill
node" in three places will mostly comply via natural compliance, not
because of the regex; and a model that misbehaves will phrase it
differently than the five exact patterns matched here. Either commit
to a real defense (resolve `process.pid` and refuse any `kill`/
`Stop-Process` whose target set includes self) or drop the regex layer
entirely and keep just the prompt guidance.

**The two prompt copies will drift.** `session/system.ts:62-67` and
`tool/shell/shell.txt:11-15` are byte-identical today and will diverge
silently the next time someone edits one. Extract a single constant
and inject it into both surfaces.

**No tests.** A safety filter without a test that exercises the
allow/deny boundary is a regression waiting to happen — the next
refactor of the regex set will silently broaden or narrow the gate.
At minimum: one positive case per pattern, plus negatives for
`killall nodejs`, `kill 1234` (a numeric PID that *is* node), and a
benign `node --version`.

**Error message echoes the command back to the model.** Generally
fine, but the `params.command` here is untrusted text — make sure the
formatter at the throw site doesn't interpret control characters when
this surfaces in the TUI. Looks safe today; flagging for awareness.

I'd merge a v2 that extracts the warning constant, adds tests, and
either deepens or deletes the regex layer.
