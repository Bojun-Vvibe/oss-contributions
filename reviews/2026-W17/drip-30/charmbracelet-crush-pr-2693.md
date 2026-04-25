# charmbracelet/crush#2693 — fix(mcp): expand environment variables in stdio MCP server args

- PR: https://github.com/charmbracelet/crush/pull/2693
- Author: octo-patch
- +11 / -1
- Base SHA: `31025a1addb9124ceadcc80f346a9e834fcf34a4`
- Head SHA: `de47207a4e5c176876a13a390c439f97b8a075e5`
- State: OPEN

## Summary

In `createTransport` for stdio MCP servers, every `m.Args[i]` is
now passed through `resolver.ResolveValue(arg)` so that
`${HOME}/.config/foo` and `$API_KEY`-style placeholders in MCP
config args expand the same way they already did for the `env`
block. Resolution failures are logged and the original (unresolved)
arg is kept, so a misconfigured arg doesn't kill the whole MCP
server launch.

## Specific findings

- `internal/agent/tools/mcp/init.go:447` (head SHA `de47207a`) —
  the new pre-loop replaces a single
  `exec.CommandContext(ctx, home.Long(command), m.Args...)` with
  a `resolvedArgs := make([]string, len(m.Args))` and a per-arg
  `resolver.ResolveValue(arg)` call. The fall-through behavior
  on error (`slog.Error(...); resolvedArgs[i] = arg; continue`)
  is the right policy — log loudly, preserve user input, let the
  child process fail with a clearer error than "config didn't
  load."
- The `home.Long(command)` call on the *command* itself is
  unchanged — only `Args` get the new `ResolveValue` treatment.
  This is asymmetric: a user with `command:
  "${HOME}/bin/my-mcp-server"` would have that expanded by
  `home.Long` (which handles `~` and possibly `$HOME`), but a
  more general `${API_KEY}` in the command path would not.
  Probably fine — commands are paths, args carry more
  configuration — but worth a comment explaining the split.
- The `m.ResolvedEnv()` call at the next line shows that the
  env block already goes through the resolver. So this PR is
  literally bringing args to parity with env, which is the
  right consistency fix.
- `slog.Error("Error resolving MCP arg", ...)` includes the
  failing arg in the log line. If args ever carry secrets
  (`--token=${API_KEY}`), that would leak the literal
  `${API_KEY}` placeholder (not the resolved secret) into logs —
  fine, that's the placeholder name only.

## Risks

- No test added in the diff slice reviewed. The behavior change
  is small but worth a 1-line table-driven test case in
  `init_test.go` (or wherever `createTransport` is tested) to
  confirm `${HOME}` expansion in args round-trips and a
  resolver error preserves the literal arg.
- If `resolver.ResolveValue` ever has a side effect (e.g., logs
  every lookup at info level) the per-arg loop could flood logs
  for MCP servers with many args. Probably not; the existing env
  resolver already iterates similarly.

## Verdict

`merge-after-nits`

## Rationale

Tiny, mechanically obvious parity fix that brings stdio MCP args
in line with the env block's existing variable-expansion behavior.
The fall-open-on-error policy is correct (log and keep going).
Just add a test and consider a one-line comment explaining why
the command path uses `home.Long` while args use `ResolveValue`.

## What I learned

When a config struct has two adjacent fields that both look like
"things the user can write `${...}` in" (here: `Command`+`Args`
and `Env`), they should always go through the same resolver in
the same order. Drift between them is a steady source of
"why does it work in env but not args" bug reports.
