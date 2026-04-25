# charmbracelet/crush #2693 — fix(mcp): expand environment variables in stdio MCP server args

- **Repo**: charmbracelet/crush
- **PR**: [#2693](https://github.com/charmbracelet/crush/pull/2693)
- **Head SHA**: `de47207a4e5c176876a13a390c439f97b8a075e5`
- **Author**: octo-patch
- **State**: OPEN (+11 / -1)
- **Verdict**: `merge-after-nits`

## Context

Fixes #2684. Stdio MCP server configs allow `command` and
`args`; `command` already runs through
`resolver.ResolveValue` (so `$HOME/bin/foo` becomes
`/Users/x/bin/foo`), but `args` were passed verbatim. So a
config like:

```json
"vestige": {
  "type": "stdio",
  "command": "vestige-mcp",
  "args": ["--data-dir", "$VESTIGE_DB"]
}
```

launched `vestige-mcp --data-dir '$VESTIGE_DB'` with the
literal dollar-sign string instead of the resolved value.

## Design

10-line patch in
`internal/agent/tools/mcp/init.go:447-460`. Builds a
`resolvedArgs []string` parallel to `m.Args` and substitutes
each entry through the same `resolver` that already handles
the `command` field:

```go
resolvedArgs := make([]string, len(m.Args))
for i, arg := range m.Args {
    resolved, resolveErr := resolver.ResolveValue(arg)
    if resolveErr != nil {
        slog.Error("Error resolving MCP arg", "error", resolveErr, "arg", arg)
        resolvedArgs[i] = arg
        continue
    }
    resolvedArgs[i] = resolved
}
cmd := exec.CommandContext(ctx, home.Long(command), resolvedArgs...)
```

Then `exec.CommandContext` is called with `resolvedArgs...`
instead of `m.Args...`. Pre-allocating `resolvedArgs` with
`make([]string, len(m.Args))` is the right shape (no append /
realloc), and the indexed assignment lets the error-path
fallback (use the original `arg`) cleanly co-exist with the
success path.

The error policy — log + fall back to literal — matches what
shells do for unset variables in single-pass expansion, and
matches what `command` field does: `home.Long(command)` only
expands `~`, but `resolver.ResolveValue` is presumably called
on `command` somewhere upstream of this function (otherwise
`$VAR` in `command` would also be broken, and the bug report
would say so).

## Risks

- **Fallback-to-literal on resolve error is potentially
  dangerous.** If `$SECRET_TOKEN` fails to resolve (e.g., the
  resolver hits a transient backend error), the subprocess
  gets the literal string `$SECRET_TOKEN` as an argument. For
  a `--api-key $SECRET_TOKEN`-style argument, that means the
  MCP server starts with the literal `$SECRET_TOKEN` as its
  API key, which will fail in a confusing way (the MCP server
  might log the literal string in error messages). A
  fail-loud policy ("if any arg fails to resolve, abort the
  whole transport setup with an error") would be safer for
  the credentials-in-args case.
- **No new test.** A unit test that builds an
  `MCPConfig{Args: []string{"--data-dir", "$X"}}`, sets
  `X=/tmp/foo` in the resolver, and asserts
  `cmd.Args[2] == "/tmp/foo"` would lock this in for ~10
  lines. The repo already has init_test.go-style coverage
  for the transport factory.
- **Symmetry with `Env` handling.** `cmd.Env =
  append(os.Environ(), m.ResolvedEnv()...)` on line 21 of
  the surrounding diff suggests the env path uses a
  pre-resolved field (`ResolvedEnv()`) rather than a
  call-site resolve. If config schema has a `ResolvedArgs()`
  method available, using it would match the existing
  pattern; if not, the inline loop is fine but worth a
  one-line comment noting the asymmetry.
- The `slog.Error` log message ("Error resolving MCP arg")
  doesn't include which MCP server's args are being
  resolved. For a config with multiple stdio servers, that
  context (`m.Name` or whatever the field is called) makes
  the log line actionable.

## Suggestions

- **Tighten the error policy**: rather than fall back to the
  literal string, return the resolve error from
  `createTransport` and let the caller mark the MCP server as
  failed. This matches "fail-loud on credential resolution
  failure" — much safer for `$API_KEY`-style args.
- Add the MCP server name to the `slog.Error` log line:
  `slog.Error("...", "server", m.Name, "error", resolveErr, "arg", arg)`.
- Add a unit test exercising both success and resolve-failure
  paths.
- Consider adding `m.ResolvedArgs()` as a config-level
  helper if `m.ResolvedEnv()` already exists — keeps the
  resolve logic in one place rather than scattered across
  `createTransport`.

## Verdict reasoning

`merge-after-nits`. The fix is correct and the bug is real
(arg-vs-command asymmetry on the same resolver), but the
fallback-to-literal-string policy on resolve failure is a
real footgun for credential-bearing args. Tightening the
error path before merge is worth the round-trip; the test
and log-context tweaks are nice-to-haves.

## What I learned

The "command resolves env vars but args don't" asymmetry is
a recurring pattern in subprocess-spawning code — usually
because `command` gets a `~` expansion via a path helper
(`home.Long`) and someone assumes that's "the same as" env
expansion. They're different operations: `~` expansion is
literal-prefix replacement, env expansion is variable
substitution anywhere in the string. Treating them as the
same thing means args that look like `--data-dir $HOME/foo`
**won't** get the `$HOME` expanded but **will** confuse the
maintainer who reads the code. The clean fix is to push both
through the same `ResolveValue` pipeline so one mental model
governs the whole transport-spec → exec-args pipeline.
