# charmbracelet/crush PR #2778 — feat(hooks): allow hook names

- **Head SHA**: `928c8f4bbde2d28d4df52ee9422db94a206e8491`
- **Scope**: `internal/config/config.go`, `internal/hooks/runner.go` + test

## Summary

Adds an optional `Name` field to `HookConfig` (`internal/config/config.go:377`) and a `DisplayName()` accessor (`internal/config/config.go:387-393`) that falls back to `Command` when `Name` is empty. The runner now uses it at `internal/hooks/runner.go:113`:

```go
Name: h.DisplayName(),
```

Tests at `internal/hooks/hooks_test.go:469-498` cover both branches (named and unnamed).

## Comments

- Pure additive, backward-compatible: existing configs without `Name` keep showing the command verbatim, matching prior TUI behavior.
- JSON schema annotation is consistent with the surrounding fields (`description=...`).
- `DisplayName()` is a value method on `*HookConfig` — fine; consider value receiver since it doesn't mutate, but Go style here is split and the existing `TimeoutDuration()` pattern matches.
- One observation: there's no validation against duplicate `Name` values across hooks. If two hooks share a name, the TUI will surface ambiguous rows. Not a blocker — the existing dedupe by command is downstream of this — but worth a follow-up.
- Test names are descriptive and the parallel sub-tests are well structured.

## Verdict

`merge-as-is`
