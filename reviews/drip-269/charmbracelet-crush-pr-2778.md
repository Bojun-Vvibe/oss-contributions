# charmbracelet/crush #2778 — feat(hooks): allow hook names

- URL: https://github.com/charmbracelet/crush/pull/2778
- Head SHA: `928c8f4bbde2d28d4df52ee9422db94a206e8491`
- Author: @BrunoKrugel
- Stats: +43 / -1 across 3 files

## Summary

Closes #2763. Adds an optional `Name` field to `HookConfig` and a
`DisplayName()` accessor that returns `Name` when set, otherwise falls
back to `Command`. The runner replaces the bare `Name: h.Command`
assignment with `Name: h.DisplayName()` so the TUI can show
human-readable identifiers (e.g. "format-on-save") instead of raw shell
commands like `gofmt -w .` cluttering the UI.

## Specific feedback

- `internal/config/config.go:377` — new field carries
  `json:"name,omitempty"` and a jsonschema description. `omitempty` is
  the right choice; existing configs without `Name` round-trip
  byte-identically.
- `internal/config/config.go:387-394` — `DisplayName()` is a one-liner
  fallback. Implementation matches the contract in the doc comment.
  Receiver is `*HookConfig` (pointer), consistent with the existing
  `TimeoutDuration()` method on the same type.
- `internal/hooks/runner.go:113` — single-character call-site change
  from `h.Command` → `h.DisplayName()`. Clean, surgical.
- `internal/hooks/hooks_test.go:469-498` — both branches covered:
  "Name is used when set" (`my-hook`) and "Command is used when Name is
  empty". Both cases assert `DecisionAllow`, hook count, and the
  resulting `Name`. Solid coverage.
- Before/after screenshots in the PR body show the actual TUI win:
  cluttered command strings replaced by short labels.
- Nit: when `Name` is set but is an extremely long string the TUI may
  still truncate awkwardly. Worth a follow-up to enforce a sane max
  length (or at least document it in the jsonschema description), but
  not blocking.
- The PR notes the "I have created a discussion that was approved by a
  maintainer" checkbox is unchecked. Since this is described as `feat`
  (not `fix`) and the CONTRIBUTING.md gates new features behind a
  discussion, maintainers may want to confirm before merge. Mentioning
  for transparency.

## Verdict

`merge-after-nits` — code is clean and tested. The unchecked discussion
checkbox is the only governance ask; the technical content is ready.
