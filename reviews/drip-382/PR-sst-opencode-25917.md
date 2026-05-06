# sst/opencode PR #25917 — fix(shell): advertise actual default timeout in tool description

- URL: https://github.com/anomalyco/opencode/pull/25917
- Head SHA: `78eacba8fce1509cb42c860a0a7e54ef46201f29`
- Size: +5 / -4

## Summary

Plumbs `DEFAULT_TIMEOUT` through the shell tool's prompt-rendering `Limits`
record so the tool description text reflects the *actual* configured timeout
when the user overrides it via `OPENCODE_EXPERIMENTAL_BASH_DEFAULT_TIMEOUT_MS`,
instead of the previously-hardcoded `120000ms (2 minutes)` literal that was
duplicated across the bash, powershell, and cmd command sections.

## Specific findings

- `packages/opencode/src/tool/shell/prompt.ts:17-21` — `Limits` gains a third
  numeric field `defaultTimeoutMs: number` alongside the existing `maxLines`
  and `maxBytes`. Pure additive type extension, no consumers broken (the only
  external caller is the new `render()` site at `shell.ts:588`).
- `packages/opencode/src/tool/shell.ts:588` — call site widens
  `ShellPrompt.render(name, process.platform, limits)` to
  `ShellPrompt.render(name, process.platform, { ...limits, defaultTimeoutMs: DEFAULT_TIMEOUT })`.
  Spreading inline rather than threading `defaultTimeoutMs` through the
  `Truncate.limits()` accessor is the right call — `Truncate` is responsible
  for line/byte truncation and has no business knowing about the shell's
  timeout policy. Coupling them would muddy the abstraction.
- `prompt.ts:106, 152, 202` — three identical edits replace the literal
  `120000ms (2 minutes)` with `${limits.defaultTimeoutMs}ms` in the bash,
  powershell, and cmd sections. Confirmed all three are reachable: `Shell.name`
  selects between the three in `shell.ts` and `bashCommandSection` /
  `powershellCommandSection` / `cmdCommandSection` are dispatched by the
  outer `render()`. No fourth section was missed.

## Notes

- The new rendered string drops the parenthetical "(2 minutes)" gloss that
  used to follow the literal — when `DEFAULT_TIMEOUT` happens to be `120000`
  the description now reads `…time out after 120000ms.` instead of
  `…time out after 120000ms (2 minutes).`, slightly less self-documenting for
  the model. A trivially better template would be
  `${limits.defaultTimeoutMs}ms (~${Math.round(limits.defaultTimeoutMs/1000)}s)`,
  but this is a nit.
- No test coverage of the rendered prompt for the override case. A one-liner
  that calls `ShellPrompt.render("bash", "linux", { maxLines: 1, maxBytes: 1, defaultTimeoutMs: 30000 })`
  and asserts the rendered string contains `30000ms` would pin the contract;
  worth adding but not a blocker.
- The PR body says `bun turbo typecheck` passes and existing shell test
  suites are unchanged — consistent with a 5-line change that only affects
  the prompt-text rendering.

## Verdict

`merge-after-nits`
