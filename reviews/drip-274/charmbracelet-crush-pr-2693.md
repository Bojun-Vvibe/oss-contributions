# Review: charmbracelet/crush #2693 — fix(mcp): expand env vars in stdio MCP server args

- Repo: charmbracelet/crush
- PR: #2693
- Head SHA: `de47207a4e5c176876a13a390c439f97b8a075e5`
- Author: octo-patch
- Size: +11 / -1 across 1 file

## What it does
Resolves config variable references (e.g. `$HOME`, custom resolver values)
inside `m.Args` for stdio MCP server transports, mirroring how `command`
and `env` are already resolved. Without this, an arg like `--config=$HOME/foo`
was passed verbatim and the spawned process had to expand it itself.

## File-level notes

**`internal/agent/tools/mcp/init.go` @ L447–460 (head `de47207`)**
```go
- cmd := exec.CommandContext(ctx, home.Long(command), m.Args...)
+ resolvedArgs := make([]string, len(m.Args))
+ for i, arg := range m.Args {
+     resolved, resolveErr := resolver.ResolveValue(arg)
+     if resolveErr != nil {
+         slog.Error("Error resolving MCP arg", "error", resolveErr, "arg", arg)
+         resolvedArgs[i] = arg
+         continue
+     }
+     resolvedArgs[i] = resolved
+ }
+ cmd := exec.CommandContext(ctx, home.Long(command), resolvedArgs...)
```

- Failure handling falls back to the unresolved arg, which preserves prior
  behavior on resolver errors. Good defensive choice.
- The `slog.Error` line will log the raw arg value, which may contain
  secrets if a user templated `$API_KEY` and resolution failed. Consider
  logging only the index or a redacted preview, e.g. `slog.Error(...,
  "arg_index", i)` instead of the full arg string.
- No `home.Long(...)` is applied to args — only `command` gets it. If the
  resolver doesn't expand `~`, args like `~/foo` will remain literal.
  Probably fine since `home.Long` is path-specific and args may be
  non-paths, but worth confirming the resolver covers the same surface
  users expect.
- No new test. The existing test file `init_test.go` (if present) would
  benefit from a table-driven case asserting `${VAR}` in args is expanded.

## Risks
- Low. Single transport path, fail-open on errors.
- Logging concern (above) is the only real risk surface.

## Verdict: `merge-after-nits`
Redact the arg value in the error log, add a smoke test, and this lands.
