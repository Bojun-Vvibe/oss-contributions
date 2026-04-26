# Review — charmbracelet/crush#2693: fix(mcp): expand environment variables in stdio MCP server args

- **Repo:** charmbracelet/crush
- **PR:** [#2693](https://github.com/charmbracelet/crush/pull/2693)
- **Author:** octo-patch (Octopus)
- **Head SHA:** `de47207a4e5c176876a13a390c439f97b8a075e5`
- **Size:** +11 / −1 across 1 file (`internal/agent/tools/mcp/init.go`)
- **Verdict:** `merge-after-nits`

## Summary

Closes a config gap in the stdio MCP transport: prior to this PR, `command` was passed through `home.Long(command)` (which expands `~` and presumably `${VAR}`-style references via `resolver.ResolveValue`), but the `args` array was passed verbatim. That meant a config like `args: ["--token", "$MY_TOKEN"]` would invoke the MCP server with the literal string `$MY_TOKEN` rather than the resolved value. This PR loops over `m.Args`, calls `resolver.ResolveValue(arg)` on each, and falls back to the original arg on resolution error (with a `slog.Error`).

## Technical assessment

The diff is the minimum sufficient change at `init.go:447-462`:

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

This matches the same `resolver` instance already used to resolve `command` at the prior single line of the file (the diff shows `command` was previously already going through some resolution; this just extends parity to args). The fallback-to-original-on-error behavior is sane: a resolver error doesn't blow up the whole server start, it just logs and uses the literal string. That's the right tradeoff — better to launch with `$MY_TOKEN` literal and let the MCP server complain than to refuse to launch entirely.

## Nits worth addressing pre-merge

1. **`slog.Error` for a config-arg resolve failure may be too loud.** If the resolver legitimately can't resolve a literal arg (e.g. user wrote `args: ["--flag-with-$-sign-but-not-a-var"]`), this will log at error level on every server start. Consider `slog.Warn` instead, or only log at error when the arg contains a `$` or `${` prefix (i.e. clearly intended as a variable reference). One-line refinement.

2. **No test added.** A 1-file 11-line MCP transport change without a test is the kind of thing that regresses silently when someone refactors `resolver.ResolveValue`. Recommend a test in the same package that constructs an `MCPConfig{Args: []string{"--token", "$MY_TOKEN"}}` with a fake `resolver` returning `"abc123"` for `$MY_TOKEN`, and asserts that the resulting `cmd.Args` (or, since `exec.CommandContext` is used directly here, a refactor to allow injection) contains `"abc123"`. If the function is currently untestable due to direct `exec.CommandContext` usage, that's a separable refactor worth doing in a follow-up.

3. **Symmetry with `m.ResolvedEnv()`.** At `init.go:464`, `cmd.Env = append(os.Environ(), m.ResolvedEnv()...)` — the env block already goes through a `Resolved*` helper. The args are now resolved inline rather than through a `m.ResolvedArgs()` helper. For consistency, consider extracting `func (m MCPConfig) ResolvedArgs(resolver config.Value) ([]string, error)` matching the `ResolvedEnv` shape. That moves the loop into the config struct's own methods (where it's testable in isolation) and keeps `createTransport` thin.

4. **Error aggregation.** Each arg resolves independently and logs independently. If a config has 5 args and 3 fail to resolve, you'll get 3 separate `slog.Error` entries. Could aggregate into a single log line with `failed_args=[...]` for easier ops parsing. Minor.

5. **Doc update.** If the docs for MCP stdio config don't already mention "args support env-var expansion", they should now. Likely lives in `docs/configuration/mcp.md` or similar — not in this diff window but worth a follow-up.

## Verdict rationale

`merge-after-nits`. The fix is correct, minimal, and addresses a real config-symmetry gap (env vars get resolved in `command` and in `env`, but were not in `args`). The lack of a test and the slog level are real but low-cost asks. The symmetry refactor (item 3) is a nice-to-have that can be deferred. Once a test lands or item 1's slog adjustment is made, this is mergeable.
