# charmbracelet/crush PR #2782 — fix(config): restore full shell expansion in MCP config values

- URL: https://github.com/charmbracelet/crush/pull/2782
- Head SHA: `40684228138303a922ff71a8f39dfe85fad30572`
- Author: meowgorithm (Christian Rocha)

## Summary

Re-introduces full shell-style expansion (`$VAR`, `${VAR:-default}`,
`$(command)`, quoting, nesting) on every user-supplied string field in the
MCP config block: `command`, `args`, `env`, `headers`, and `url`. The
expansion is routed through Crush's embedded shell so the same syntax
works on Linux/macOS/Windows. Unset variables become a hard error; opt
into a default with `${VAR:-fallback}`.

In `internal/agent/tools/mcp/init.go::createTransport`, every previously
direct-read config field (`m.Args`, `m.ResolvedEnv()`, `m.URL`,
`m.ResolvedHeaders()`) is now routed through resolver-aware methods
(`ResolvedArgs`, `ResolvedEnv`, `ResolvedURL`, `ResolvedHeaders`) that
take a `config.VariableResolver`. Added test coverage in
`init_test.go::TestCreateTransport_URLResolution` exercising both http and
sse transports, success and failure cases for `$VAR`/`$(cmd)` forms.

## Summary of Review

This restores parity that was lost when MCP config moved off the global
env-resolver. Three things to verify before merge:

1. **Hard-error on unset is a behavior change.** Existing users who had
   `"$OPTIONAL_TOKEN"` in their config and nothing in their env would
   previously get the literal string `$OPTIONAL_TOKEN`; now they get a
   startup failure. The README change covers the migration but call it
   out in CHANGELOG so users see it on upgrade.
2. **`$(command)` on Windows.** README claims "the same syntax works on
   all supported systems, including Windows." Verify the embedded shell
   actually wires `$(cat /path)` to a Windows-equivalent — on Windows
   there's no `cat` in PATH by default. Either restrict `$(...)` to
   POSIX-named shells or document the platform caveat. The doc as
   written ("file-based secrets like work out of the box") has a stray
   word and a typo near the inline code (`"$TOKEN"\`` and
   `` `"$(cat /path/to/secret/token)"`` `` have an unbalanced backtick).
3. **`shellResolverWithPath` test helper** in `init_test.go` correctly
   notes that `$(cat)`/`$(echo)` need `PATH` set in the test process. CI
   on Windows runners will fail here unless the equivalent stub is
   provided — make sure the test is gated or skipped on `runtime.GOOS ==
   "windows"`.

The transport-layer test (`TestCreateTransport_URLResolution`) is well
structured: it pins that `m.URL` goes through the same resolver seam as
the other fields, which is exactly the kind of regression that would be
silent at the config layer.

## Verdict

`merge-after-nits` — the README has a typo/unbalanced backtick to fix,
the unset-var hard-error needs a CHANGELOG callout, and the Windows
`$(...)` claim needs verification or a documented caveat.
