# ollama/ollama#15805 — Add `ollama launch qwen` support for Qwen Code CLI

- PR: https://github.com/ollama/ollama/pull/15805
- Author: B-A-M-N
- +579 / -2
- Base SHA: `ea01af6f76fc03ba737aec1f9a49e82f6063bab1`
- Head SHA: `6469883f20524835e3ef685d365ec79d169ff483`
- State: OPEN

## Summary

Adds a new `qwen` integration to `ollama launch`, plus four launch
flags that propagate to *every* integration: `--think`,
`--config-scope`, `--provider-mode`, `--experimental`. To plumb the
new flags without breaking existing integrations' constructors, the
PR introduces a `LaunchConfigurator` interface that integrations
can optionally implement to receive the full
`IntegrationLaunchRequest` before `Run`/`Edit`.

## Specific findings

- `cmd/launch/launch.go:122` (head SHA `6469883f`) —
  `IntegrationLaunchRequest` grows four fields: `Think`,
  `ConfigScope`, `ProviderMode`, `Experimental`. Defaults at
  `cmd/launch/launch.go:309` are `"auto"`, `"user"`, `"hybrid"`,
  `false` — sensible defaults that preserve existing behavior for
  any integration that doesn't read them.
- `cmd/launch/launch.go:147` — `LaunchConfigurator` is a one-method
  interface (`ConfigureLaunch(req IntegrationLaunchRequest)`).
  This is the right escape hatch: integrations that don't care
  (vscode, opencode) implement nothing; new ones (qwen) opt in.
  The cast at `launch.go:369` (`if lc, ok :=
  runner.(LaunchConfigurator); ok { lc.ConfigureLaunch(req) }`) is
  the standard Go optional-interface pattern.
- `cmd/launch/launch.go:271` — the bare-`launch` (no integration
  name) guard now also rejects `--think`, `--config-scope`,
  `--provider-mode`, `--experimental`. Good — previously these
  would silently fall through to the TUI launcher and be ignored;
  now they error explicitly.
- The help text at `launch.go:230` adds `qwen  Qwen Code CLI
  (experimental)`. The `(experimental)` annotation is consistent
  with the new `--experimental` flag's existence; users who want to
  use it will need to pass `--experimental` explicitly (verify
  in the qwen integration file, not visible in this slice).
- The `cmd/launch/launch_feature_test.go` file is new
  (truncated in the diff slice reviewed, +92 lines). Reviewers
  should confirm it covers (a) defaults flow through correctly,
  (b) the bare-`launch` flag-guard rejects the four new flags,
  and (c) `LaunchConfigurator.ConfigureLaunch` is invoked exactly
  once before `Run`.

## Risks

- The four new flags are global to `ollama launch` regardless of
  whether the chosen integration understands them. A user passing
  `--think on` to the `vscode` integration will get silent no-op
  behavior. Either log a warning when an integration without
  `LaunchConfigurator` receives non-default flag values, or
  document the matrix.
- `--provider-mode hybrid|config|env` is a new tri-state knob with
  no documented semantics in the diff slice. The default `hybrid`
  is the safest default but the help string doesn't explain what
  `config` vs `env` actually do. Add inline help or a docs link.

## Verdict

`merge-after-nits`

## Rationale

Solid extension of the launch interface with a clean opt-in
mechanism (`LaunchConfigurator`) that doesn't break existing
integrations. The two nits — silent-no-op for non-aware
integrations, and the under-documented `--provider-mode` enum —
are doc/UX issues, not correctness ones. The new feature test
file is reassuring; reviewers just need to confirm its
coverage matches the change surface.

## What I learned

When adding a new optional capability to a plugin interface, an
*optional* sub-interface (Go's `if v, ok := x.(Foo); ok` pattern,
or Python's `hasattr`) is almost always better than widening the
required interface — old plugins keep compiling, new plugins
opt in.
