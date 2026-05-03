# charmbracelet/crush PR #2778 — feat(hooks): allow hook names

- Author: BrunoKrugel
- Head SHA: `928c8f4bbde2d28d4df52ee9422db94a206e8491`
- Diff: +43 / -1 across 3 files
- Files: `internal/config/config.go`, `internal/hooks/hooks_test.go`, `internal/hooks/runner.go`

## Observations

1. **`config.go:377-379` adds `Name string` to `HookConfig`** with `json:"name,omitempty"` and a JSON-schema description. Field is optional, so existing user configs continue to work without migration. Good.
2. **`config.go:387-393` adds `(*HookConfig).DisplayName()`** that returns `Name` when non-empty, falling back to `Command`. Receiver is a pointer — minor nit: the rest of `HookConfig` methods like `TimeoutDuration()` use the same pointer-receiver convention so this is consistent.
3. **`runner.go:113`** is the single behavior change: `Name: h.Command` becomes `Name: h.DisplayName()`. This is the right place — every aggregator (`agg.Hooks[i]`) goes through this loop. UI surfaces that previously displayed the raw `Command` (which can be very long shell snippets) will now show the friendly name when configured.
4. **`hooks_test.go:469-499` covers both branches**: `name field is used when set` and `command is used when name is empty`. The fallback test (`:485-498`) explicitly asserts `Hooks[0].Name == 'echo \'{"decision":"allow"}\''` — confirming back-compat with users who never set `Name`. That's the regression test I'd want.
5. **No telemetry/log impact considered**: if any structured log line elsewhere keys on `HookInfo.Name` (e.g. for hook-level metrics), the cardinality just changed from "command-string" to "user-supplied label". Probably an improvement, but worth a `grep -r 'HookInfo' internal/` to be sure nothing assumes the prior shape (e.g. de-dup by `Name`).
6. **Schema regeneration**: the `jsonschema:"description=..."` tag on a new field typically requires regenerating any committed JSON schema docs. If this repo ships a `schema.json` for editor IntelliSense, it may need a refresh in the same PR.

## Verdict: `merge-after-nits`

Small, focused, well-tested. Two follow-ups before merge: (a) regenerate any committed JSON schema for the config, and (b) confirm no log/metric path was implicitly grouping by the previous `Name == Command` convention.
